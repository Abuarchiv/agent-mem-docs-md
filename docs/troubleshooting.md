# Troubleshooting

Every message agent-mem can show, what causes it, and what to do.

## Start here

```sh
agent-mem doctor
```

Every check prints `✓`, `!` with the next step, or `✗`. Most problems below show up there first. For any error
in a command, `AGENT_MEM_DEBUG=1 agent-mem <command>` prints the full traceback.

## Nothing is recorded

<details>
<summary>The panel never appears and <code>agent-mem status</code> shows no sessions</summary>

**Cause:** the hooks are not registered, not trusted, or point at a missing binary.

1. `agent-mem doctor` lists, per agent, whether hooks are registered, whether they match this release, and the
   command they run. Check that the path exists.
2. **Codex:** start `codex`, run `/hooks`, trust the agent-mem hooks.
3. **Claude Code:** `/hooks` shows what is registered. Plugin and `setup --write` both on? Keep one.
4. **Copilot:** use `agent-mem setup copilot --write`; plugin-defined `preToolUse` hooks do not fire.
5. Rewrite them: `agent-mem setup <agent> --write`.

</details>

<details>
<summary>One project is never recorded</summary>

Check, in this order: `agent-mem status` (paused?), `excluded_projects` in `config.json`, a `.agent-mem.json` with
`"capture": false` in the repository root, and whether the folder is inside `~/.claude`, `~/.codex`, `~/.copilot` or
`~/.config/opencode` (never recorded). `AGENT_MEM_DISABLE=1` in the agent's environment turns everything off.

</details>

<details>
<summary><code>Executable not found in $PATH: "agent-mem"</code> or <code>command not found</code> in every hook</summary>

**Cause:** the hooks call a command that is gone or not on the agent's `PATH`. Typical after `uv tool uninstall`
without `agent-mem uninstall all`, or with the Claude Code plugin but no installed command.

**Fix:** install the command (`uv tool install "agent-mem[semantic,observer] @ git+https://github.com/Abuarchiv/agent-mem"`)
and run `agent-mem setup all --write`, which writes absolute paths. To remove agent-mem instead, see
[Uninstall](getting-started/upgrade-uninstall.md#uninstall).

</details>

<details>
<summary><code>agent-mem cannot save events: they wait in the spool. Run agent-mem doctor.</code></summary>

**Cause:** events could not reach the database for more than 30 seconds: a lock held by another process, a
pending migration, or a damaged database.

**Fix:** `agent-mem index` replays the spool (and migrates if needed). `agent-mem status` shows "Not saved yet: N
events". If it stays, `agent-mem doctor` names the database problem.

</details>

<details>
<summary><code>agent-mem cannot save events (…). This session is not being recorded.</code></summary>

**Cause:** neither the database nor the spool can be written. Usually a full disk or a data folder without write
permission.

**Fix:** free disk space; `agent-mem doctor` checks space and permissions (`doctor --fix` makes the folder
private and writable by you).

</details>

<details>
<summary><code>agent-mem cannot use its database (…). What happens now is kept and saved later.</code></summary>

**Cause:** the database failed (damaged, schema newer than this release, unexpected error). Events go to the spool.

**Fix:** `agent-mem doctor`. A damaged database: `agent-mem index` moves it aside as `memory.db.corrupt-<time>`
and starts a new one. A newer schema: `agent-mem upgrade`.

</details>

<details>
<summary><code>status</code> says "could not be read: N files"</summary>

Hook payloads that were not valid JSON, lacked a session id or a required field, or exceeded `limits.stdin_bytes`
are noted in `quarantine/` instead of being stored. `logs/hook.log` names the reason. A few after an agent update
can mean the agent changed its payload; report it. Notes are deleted after 30 days.

</details>

## No facts

<details>
<summary>Requests stay "waiting for facts"</summary>

`agent-mem observe --status` shows the queue. Then:

| Panel or status says | Do |
| --- | --- |
| `Facts: off, codex is not installed` | install the agent's CLI, then `agent-mem observe --reprocess-raw` |
| `Facts: paused, claude is not logged in` | log in once in that CLI; requests wait and are retried within the hour |
| `Facts: paused until 14:20, the usage limit of claude is reached` | wait; requests are processed after the pause |
| nothing, but requests wait | `agent-mem observe` runs a pass now and prints what happened |
| `No facts for N request(s) (last error: …)` | `agent-mem observe --status`, then `agent-mem observe --retry-failed` |

The daily limit (`observe.daily_limit`, 300 calls) holds requests until midnight UTC.

</details>

<details>
<summary><code>Writing facts is failing: …</code> at session start</summary>

The observer's last run ended with an error. The message names it; `logs/observe.log` has the run. `agent-mem
observe` runs it in the foreground. "facts unavailable, turns kept raw" means mem0 could not be built (an extra or
the embedder is missing): `agent-mem upgrade --extras semantic observer`.

</details>

<details>
<summary>OpenCode requests never get facts</summary>

They go to `opencode run` with the free `opencode/big-pickle` model. Check that `opencode` is on `PATH` for the
observer and that `agent-mem doctor` does not name it as failing or paused; `observe.opencode_model` picks another
model. Without OpenCode installed a request is saved word for word.

</details>

## Search and context

<details>
<summary>The agent gets no context for a prompt</summary>

Only confident matches are injected. Check what exists with `agent-mem search "<key words>"`. A hit below score
0.35, or covering less than half of the words, is left out on purpose. Lower `retrieval.inject_min_score` or
`retrieval.min_term_coverage` for more (and noisier) context.

</details>

<details>
<summary><code>Search by meaning: unavailable</code></summary>

| Cause | Fix |
| --- | --- |
| the `[semantic]` extra is missing | `agent-mem upgrade --extras semantic` |
| the model is not downloaded | `agent-mem models install` |
| macOS on Intel | no fastembed wheels there; word search and the graph still work |
| the model failed to load | `agent-mem doctor` names the error; delete `models/` and run `agent-mem models install` |

</details>

<details>
<summary><code>topic links: en_core_web_sm missing</code></summary>

Facts work without topic links. Set `observe.entity_model` to the language of your facts (`de_core_news_sm`,
`en_core_web_sm`, …) and run `agent-mem models install`.

</details>

<details>
<summary>The briefing is too long or too short</summary>

`budgets.session_start` (800) sets it. Claude Code receives at most 2,400 tokens per block regardless; Codex
7,800. See [Session briefing](using/briefing.md).

</details>

## Viewer

<details>
<summary><code>Cannot start the viewer on 127.0.0.1:37778: …</code></summary>

Another program uses the port. Set `"viewer": {"port": <free port>}` in `config.json`. If another agent-mem viewer
already answers there, `agent-mem serve` says so instead and exits 0.

</details>

<details>
<summary>The page says "agent-mem was updated"</summary>

The viewer noticed new code after an upgrade and restarts; the page reloads in 3 seconds. If it stays, run
`agent-mem view`, which starts it.

</details>

<details>
<summary>The page says "agent-mem cannot read its memory"</summary>

The database could not be opened (locked for long, damaged, being migrated). The page retries every 10 seconds.
Run `agent-mem doctor`.

</details>

<details>
<summary>"Forbidden: Open http://localhost:37778/ instead"</summary>

The viewer answers only to `localhost:<port>` and `127.0.0.1:<port>`. Another host name pointing at your machine is
refused (protection against DNS rebinding).

</details>

## Commands

| Message | Cause | Fix |
| --- | --- | --- |
| `agent-mem: the database is busy (database is locked)…` | another agent-mem process holds a long write | try again shortly |
| `agent-mem: the database is damaged (…)` | integrity check failed | `agent-mem index` keeps it as `memory.db.corrupt-<time>` and starts a new one |
| `agent-mem: database schema N is newer than supported M…` | a newer release wrote the database | `agent-mem upgrade` |
| `agent-mem: missing module X…` | an extra is not installed | `agent-mem upgrade --extras semantic observer` |
| `agent-mem: <file> cannot be read as JSON (…). Nothing was changed…` | setup found an agent config it cannot parse | fix the file (a trailing comma, a comment), run setup again |
| `Refusing to delete … without --yes.` | purge in a script | add `--yes` |
| `<dir> is not a project root; it lies in project …` | `purge --project` with a folder inside a project | pass the project's root |
| `No recorded project at <dir>.` | `--project` names a folder agent-mem never saw | check the path |
| `duration must look like 30m, 2h or 1d` | `pause --for` | use a number with `m`, `h` or `d` |
| `This copy was not installed with uv tool…` | `upgrade` on a pipx or pip install | upgrade the way you installed, then `agent-mem doctor --fix` |
| `Upgrade failed; uv names the cause above.` | uv could not install | read uv's message above it |

## Messages an agent may see

```text
agent-mem: memory is busy or unavailable (…); try again, or ask the user to run `agent-mem doctor`.
agent-mem: the memory database is damaged (…); ask the user to run `agent-mem doctor`.
M7 belongs to another project; the user can delete it with `agent-mem purge --id M7`.
Stored M9; M3 was not superseded (not an agent memory of this project).
```

> [!TIP]
> Logs: `logs/hook.log` (one JSON line per hook problem) and `logs/observe.log` (one line per observer run) in the
> data folder. They never contain prompt text.

---

← [Token Accounting](architecture/token-accounting.md) · [Contents](../README.md#contents) · [Measurements](measurements.md) →
