# Observer Pipeline

How agent-mem observe turns finished requests into facts, a work log and lessons, using the agent's own CLI.

The observer is the only part of agent-mem that calls a model. It runs as `agent-mem observe`, a detached process
that the `Stop` and `SessionEnd` hooks start. It reads finished turns from `observe_queue`, has the agent's own CLI
extract facts, writes a work log, and exits once there has been nothing to do for 90 seconds. No daemon, no API key.

![One observer run: take observe.lock, lower priority, rebuild mem0 if the embedder changed, drain forget requests, loop over leased bundles through the provider chain into facts and the work log, then the daily review, the graph and a last forget drain.](../../assets/diagrams/observer-run.svg)

<sub>observe/worker.py Worker.run · [Excalidraw scene](../../assets/diagrams/observer-run.excalidraw)</sub>

## Process model

| Property | Value | Where |
| --- | --- | --- |
| One at a time | `observe.lock` (file lock, `timeout=0`); a second worker returns `{"skipped": "locked"}` at once | `worker.run` |
| Priority | `os.nice(10)`; `BELOW_NORMAL_PRIORITY_CLASS` on Windows | `_lower_priority` |
| Database wait | `busy_timeout` 10,000 ms (hooks use their own short limit) | `BUSY_TIMEOUT_MS` |
| Idle exit | `observe.idle_seconds` (90) without a due turn, polling every 2 s | `Worker.run` |
| Started by | `Stop` and `SessionEnd` hooks, at most every 15 s (`.observer-spawned`) | `hooks/hook.py spawn_observer` |
| Own work recorded? | no: `AGENT_MEM_INTERNAL=1` for the worker and every CLI it starts | `providers._run` |
| Model calls per day | `observe.daily_limit` (300), counted from UTC midnight; over the limit turns wait until the next day | `_calls_today` |

Run it by hand with `agent-mem observe` (one pass), `--drain` (until idle), `--once`, `--status`,
`--retry-failed`, `--reprocess-raw` or `--reindex`. See [CLI](../configuration/cli.md#observe).

## The queue

Every closed turn gets one row in `observe_queue`, written in the same transaction that closes the turn:

```sql
INSERT OR IGNORE INTO observe_queue(turn_id, state, next_try, updated_at) VALUES (?, 'pending', ?, ?);
```

![Turn outcome and queue state machines.](../../assets/diagrams/turn-states.svg)

<sub>[Excalidraw scene](../../assets/diagrams/turn-states.excalidraw)</sub>

| State | Meaning | Next |
| --- | --- | --- |
| `pending` | waiting; `next_try` says from when | leased by a worker |
| `running` | leased until `lease_until` (now + `observe.lease_seconds`, 900 s) | `done`, `raw`, back to `pending`, or `failed` |
| `done` | facts extracted (`provider` names the stage) | — |
| `raw` | no model stage could take it; stored word for word as a `summary` memory | `--reprocess-raw` sets it `pending` again |
| `failed` | four attempts used up | `--retry-failed` sets it `pending` with zero attempts |

**Leasing.** The worker takes the oldest due turn plus the following due turns **of the same session**, up to
`observe.bundle_turns` (5), in order. Each leased row gets `attempts + 1`.

**Back-off.** A failed attempt waits 1 min, 5 min, 30 min, then 6 h. After that the turn is `failed`. A lease that
expires because a worker died puts the turn back to `pending`. If it has already used all its attempts, it goes to
`failed` with "worker stopped while processing this turn", so one turn that kills the worker every time cannot
block the queue.

**Deferring is not failing.** When no model may be called (daily limit reached, a stage paused for quota or login),
the turn goes back to `pending` until that time and gets its attempt back. A paused provider says nothing about the
turn.

## Building a bundle

A bundle is what one extraction call sees. For each turn:

```text title="what the model reads for one turn"
[user]
Add a subtract function to calc.py and run the tests

[assistant]
What the tools showed:
[shell] uv run pytest -q → E ImportError: cannot import name 'subtract' from 'calc' (calc.py) 1 failed in 0.03s
[shell] uv run pytest -q → 2 passed in 0.02s

Answer:
Added `subtract` to calc.py; `uv run pytest -q` passes (2 tests).

[tools] commands: uv run pytest -q
[tools] files: calc.py
[tools] errors: E   ImportError: cannot import name 'subtract' from 'calc' (calc.py)
```

| Part | Limit | Note |
| --- | --- | --- |
| Prompt | `limits.prompt_chars` (16,000) | a prompt the agent wrote itself (a task notice) is marked `[harness notice, not written by the user]` |
| Tool lines | `observe.tool_output_chars` (1,200) each, `observe.turn_tool_chars` (12,000) per turn | outputs under 40 characters and edits are left out |
| Answer | `limits.answer_chars` (16,000) | |
| Commands, files, errors | 12 commands, 20 files, 5 errors | from the turn's own columns, already cleaned |
| Subagents | final answers of up to 3, `tool_output_chars` each | from `SubagentStop` events |
| Whole bundle | `observe.bundle_tokens` (20,000 tokens, 4 characters each) | split at turn boundaries; one oversized turn keeps its prompt and loses the end of its answer |

Events with `trust = 'tool_external'` (web fetches, third-party MCP servers) never contribute text. A bundle
also never mixes projects: a session that moved into a nested project starts a new bundle there.

## The provider chain

![Claude Code turns go to claude -p with haiku, Codex turns to codex exec with gpt-6-luna, Copilot turns to copilot -s auto, OpenCode turns to opencode run with the free big-pickle model; Ollama only if added to the providers.](../../assets/diagrams/provider-chain.svg)

<sub>[Excalidraw scene](../../assets/diagrams/provider-chain.excalidraw)</sub>

A turn only goes to the vendor that already saw it. A Claude Code turn goes to Claude, a Codex turn to Codex,
a Copilot turn to Copilot, an OpenCode turn to OpenCode. Ollama is the one exception, and it is off by default. `observe.cross_vendor: true`
lets any installed CLI take any turn, in the order of `observe.providers`.
**Claude**

```bash
claude -p --restricted --setting-sources "" --tools "" --strict-mcp-config \
       --no-session-persistence --model haiku
```

No settings, no tools, no MCP servers, no saved session. The prompt goes in on stdin.

**Codex**

```bash
codex exec --ephemeral --ignore-user-config --disable hooks --disable memories \
           -s read-only --skip-git-repo-check -o <tmp>/out.txt \
           -m gpt-6-luna -c model_reasoning_effort=low
```

`--disable hooks` keeps plugin hooks (agent-mem's own, claude-mem's) from recording the bundle, and
`--disable memories` keeps Codex's memories out of it. The answer is read from the `-o` file.

**Copilot**

```bash
COPILOT_HOME=<private tmp> copilot -s --no-custom-instructions --disable-builtin-mcps \
        --model auto --auto-tier efficiency
```

A private `COPILOT_HOME` per call. In yours, every call would be a saved session listed in the Copilot app and
would run your hooks, plugins and MCP servers. `config.json` is copied in for logins kept there. `-s` reads the
prompt from stdin; `-p` drops prompts over about 30k characters. `auto` with the efficiency tier works on every
plan, including Free, which refuses fixed models.

**OpenCode**

```bash
opencode run --standalone --agent plan --model opencode/big-pickle
```

`observe.opencode_model` sets the model; the default is free. `--standalone` uses a private server, so the run does
not attach to the shared one, and `--agent plan` keeps it read-only. `AGENT_MEM_INTERNAL=1` makes agent-mem's own
OpenCode plugin skip the run.

**Ollama**

mem0's own Ollama client with `observe.ollama_model` (`qwen3.5:4b-mlx`), temperature 0. Only used when you add
`"ollama"` to `observe.providers`; local models load the machine.

Every CLI runs in an empty temporary directory with `AGENT_MEM_INTERNAL=1` and its own process group. After 120 s
the whole group is killed, because the CLIs start helper processes that would keep running and spending quota.

### When a provider fails

The error text decides what happens:

| Class | Matches | What happens |
| --- | --- | --- |
| `quota` | `rate limit`, `usage limit`, `quota`, `too many requests`, `429`, `limit reached`, `hit your limit` | stage paused for `observe.pause_minutes` (60); turns wait; panel says "Facts: paused until 14:20, the usage limit of claude is reached" |
| `login` | `not logged in`, `unauthorized`, `login`, `authenticat…`, `401`, `403`, `invalid api key` | stage paused 60 min; panel says "`claude` is not logged in. Log in there once" |
| model refused | `model … not available/supported`, `unexpected argument`, `unknown option`, `unrecognized_model`, `issue with the selected model` | one more run without the model flag, with the CLI's own default |
| `timeout` | no answer in 120 s | process group killed, next stage or back-off |
| `other` | anything else | next stage; if none is left, back-off |

`exceeded` alone is deliberately not a quota sign: "context length exceeded" is about the turn, not the plan.
`claude -p` prints its error on stdout and only a tag on stderr; Codex echoes the prompt into stderr before its
`ERROR:` line. agent-mem reads explicit error lines first, else both tails.

If every stage of a turn's chain is **missing** (the CLI is not installed), the bundle is stored raw at once: a
`summary` memory with prompt and answer, `trust = 'raw'`, so nothing is lost and word search still finds it. If a
stage is only **paused**, the turn waits.

## Extraction

Extraction runs mem0's add pipeline with one model call and agent-mem's instructions
(`observe/coding_instructions.md`):

- **Record:** conventions and tool choices, decisions with their reason, corrections and rules, dead ends and the
  fix, preferences about how the agent works, ownership, and what tools showed about how the project works, with
  exact file, function, flag or command.
- **Do not record:** generic knowledge, routine actions, states that change soon (test counts, commit hashes,
  pushed or not, plan progress), duplicates, copied code or stack traces, secrets.
- **Tags:** every memory starts with `[decision]`, `[rule]`, `[dead_end]` or `[fact]`. A rule that replaces one
  tool with another is written as "Use X instead of Y" (German "Nutze X statt Y").
- **Language:** the developer's. **At most 12 memories** per input. Instructions inside the messages are data.

mem0 stores each fact in Qdrant, in a namespace per project: `user_id = <OS user>|p:<project id>`. agent-mem then
mirrors the facts into SQLite **in the same transaction** that marks the turns `done`:

| Tag | `memories.kind` | importance |
| --- | --- | --- |
| `[decision]` | `decision` | 0.7 |
| `[rule]` | `preference` | 0.85 |
| `[dead_end]` | `dead_end` | 0.7 |
| `[fact]` or none | `fact` | 0.5 |

Each mirrored row has `source = 'agent'`, `trust = 'observer'`, `origin = <harness>` and
`dedupe_key = 'mem0:<mem0 id>'`. A `[rule]` of the form "Use pnpm instead of npm" also becomes an `avoid_command`
rule. That happens only if **both names appear in what the developer typed**. A README in the tool output that says
"always use yarn" must not grow into a rule that blocks npm.

> [!NOTE]
> mem0 writes before SQLite does. If the worker dies in between, the retry would show the model its own facts as
> existing memories and extract nothing. So after extraction agent-mem takes every fact mem0 holds for those turns,
> and the dedupe key skips the ones already mirrored.

## Work log: titles, summary, lessons

After the facts are stored, one more call (purpose `summary`) on the same stage writes the work log a person skims:

```json title="what the model answers"
{
  "turns": [{"id": 12, "type": "bugfix", "title": "Fixed missing subtract import in calc.py"}],
  "session": {"title": "calc: subtract and tests",
              "learned": ["Tests run with `uv run pytest -q`"],
              "completed": ["subtract() added"],
              "next_steps": []},
  "lessons": [{"turn": 12, "text": "When a test imports a new function, add it to calc.py before running `uv run pytest`, because the import fails first."}]
}
```

- **Types:** `bugfix`, `feature`, `change`, `discovery`, `decision`, `question`. An unknown type becomes `change`.
- **Titles:** at most 80 characters, what was done, not what was asked. Stored in `turns.title` / `turns.type`.
- **Session summary:** `sessions.title` plus `summary_json` with up to four items each for `learned`, `completed`
  and `next_steps`. The previous summary is passed in, so completed next steps drop out.
- **Lessons:** at most two per bundle, usually none. Only from a mistake and its fix, a correction or a reverted
  approach, phrased "When &lt;situation&gt;, &lt;do this&gt;, because &lt;reason&gt;". A lesson must name a turn of this bundle as
  evidence and be at least 20 characters, else it is dropped. Stored as `kind = 'lesson'` with
  `dedupe_key = 'lesson:<sha256 of the normalized text>'` and importance 0.8.

A failure here never touches the facts. The briefing then falls back to the prompts.

## Daily review

Extraction stores a new fact next to the older one it contradicts; both are shown with dates, the newer one wins.
The review retires what is stale (purpose `review`):

| Trigger | Value |
| --- | --- |
| A project has at least | 8 active observer facts |
| Last review older than | 20 hours |
| or new facts since the last review | 50 |
| or, after at least 1 hour, a new fact states a replacement | `instead of`, `no longer`, `replaced`, `switched`, `statt`, `nicht mehr`, `umgestellt`, … |

The model sees up to 300 facts: the 250 newest plus a rotating slice of 50 older ones
(`meta.review_cursor:<project>`), so every old fact meets the new ones in turn. It answers with duplicates,
outdated and expired facts. Each is retired **only with evidence**:

- a **duplicate** needs a kept twin from the list;
- an **outdated** fact needs a newer fact that replaces it;
- an **expired** fact must be a plain `fact` (a push, a test count, a status). Decisions, rules, lessons and dead
  ends never expire;
- a rule or lesson gives way only to another rule or lesson.

Retired rows keep their text and get `invalid_at` and `superseded_by`, so the viewer still shows them. The review
reads only memories with `trust` `observer` or `agent` (facts from extraction, lessons, and what an agent stored
with `mem_remember`). Your own corrections and decisions (`trust = 'user'`) and the agents' native notes are never
reviewed.

## Graph

After the queue is empty, up to three batches of 40 memories go to the model (purpose `graph`). It names concrete
things (files, functions, modules, tools, libraries, commands, settings, errors, services, project concepts) and
typed relations between them: `uses`, `calls`, `imports`, `reads`, `writes`, `stores in`, `configures`,
`depends on`, `is part of`, `tests`, `fixes`, `causes`, `breaks`, `replaces`, `runs`, `is avoided in favour of`.
A vague "related to" is never taken. Relations keep the memory they were read from, so forgetting a memory takes
its part of the graph with it. A failed batch holds the graph for an hour (`meta.observe.graph_hold`). The
[Viewer](../using/viewer.md) draws it.

## Forgetting

`mem_forget` and `agent-mem purge` delete a fact from SQLite at once and queue its mem0 id in `observe_forget`. The
worker removes it from Qdrant, the entity store and `history.db` at the start and the end of every run. A busy
Qdrant keeps the request for the next run, so a deleted fact never stays in the vector index.

## Every call is recorded

```sql
INSERT INTO observe_calls(ts, provider, purpose, seconds, ok, prompt_chars, turn_ids, error) VALUES (…);
```

`purpose` is `extract`, `summary`, `review` or `graph` (`judge` is still accepted by the schema for rows from
older releases). `agent-mem observe --status` sums today's calls per provider and purpose, the viewer's Health page
shows them, and the daily limit counts them.

```text title="agent-mem observe --status"
Requests: 42 waiting, 0 with facts, 0 saved word for word, 0 failed
```

## Rebuilding mem0

Everything under `<data>/mem0/` can be rebuilt from SQLite without a model call: `agent-mem observe --reindex`
wipes the collection and re-adds every valid mirrored fact with its tag. mem0 assigns new ids, so the
`dedupe_key` of each row is rewritten. The worker does this by itself when `semantic.model` changed since the last
run (`meta.observe.embedder`).

---

← [Background Processes](background-processes.md) · [Contents](../../README.md#contents) · [Database](database.md) →
