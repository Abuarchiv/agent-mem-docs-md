# CLI

Every agent-mem command and option, with real output.

```text
agent-mem [--data-dir DIR] [--version] <command> [options]
```

| Global option | Effect |
| --- | --- |
| `--data-dir DIR` | use this data folder for this command (else `AGENT_MEM_DATA_DIR`, else the system default) |
| `--version` | `agent-mem 1.0.0 (7d6cf39)`: the release and, for a git install, the commit |
| `-h`, `--help` | help for agent-mem or a command |

Errors are one line with the cause and the next step, exit code 1. `AGENT_MEM_DEBUG=1` shows the traceback. A few
examples of that line:

```text
agent-mem: the database is busy (database is locked). Another agent-mem process holds it; try again shortly.
agent-mem: the database is damaged (…). `agent-mem index` keeps it as memory.db.corrupt-<time> and starts a new one.
agent-mem: missing module mem0. Add the extras that need it: `agent-mem upgrade --extras semantic observer`.
agent-mem: database schema 9 is newer than supported 8. A newer agent-mem wrote this database; upgrade agent-mem.
```

## Overview

| Command | Does |
| --- | --- |
| [`status`](#status) | what is saved, what was learned, which jobs work |
| [`doctor`](#doctor) | check that everything is set up and working; `--fix` repairs |
| [`search`](#search) | search memory |
| [`show`](#show) | an entry by id (`T12`, `M3`) |
| [`pause`](#pause), [`resume`](#resume) | stop and restart recording |
| [`export`](#export) | memory as JSON |
| [`purge`](#purge) | delete data for good |
| [`observe`](#observe) | write facts now |
| [`index`](#index) | update the search index now |
| [`consolidate`](#consolidate) | merge duplicates and retire outdated memories now |
| [`rules`](#rules) | rules learned from corrections |
| [`lessons`](#lessons) | lines to add to AGENTS.md / CLAUDE.md |
| [`models`](#models) | show or download the search and topic models |
| [`setup`](#setup) | connect an agent |
| [`uninstall`](#uninstall) | disconnect an agent; data stays |
| [`view`](#view) | open the viewer |
| [`serve`](#serve) | run the viewer; login item |
| [`upgrade`](#upgrade) | update to the newest version |
| [`paths`](#paths) | where the data is |
| [`mcp`](#mcp) | the MCP server (your agent starts it) |
| [`eval`](#eval) | measure search quality (for developers) |
| [`hook`](#hook) | the hook entry point (your agent calls it) |

## status

```text
agent-mem status [--json]
```

```text
agent-mem · ~/Library/Application Support/agent-mem · 0.49 MB
Saved: 1 project · 46 sessions · 45 requests · 2 memories
Learned: 1 error fix · rules: 1 on, 0 suggested · fixed errors that came back: 1
Search by meaning: not indexed yet (0 entries)
Not saved yet: 0 events · could not be read: 1 file
Given to agents: today ~6,245 tokens · last 7 days ~6,245 tokens in 73 reminders (~139 per session)
Last cleanup: never
Background jobs:
  Claude Code hooks: working, last 2026-10-13 21:53
  Codex hooks: working, last 2026-10-13 21:52
  Copilot CLI hooks: working, last 2026-10-13 21:52
  OpenCode hooks: working, last 2026-10-13 21:52
  Claude Code connection: working, last 2026-10-13 21:52
```

"Not saved yet" counts events in the spool, "could not be read" files in `quarantine/`. A paused recording adds
`Recording paused until …`. A job that is failing shows its last error. `--json` prints everything as one object.

## doctor

```text
agent-mem doctor [--fix] [--json]
```

```text
✓ python: 3.13.14
✓ agent-mem command: /Users/you/.local/bin/agent-mem
✓ data folder private: ~/Library/Application Support/agent-mem
✓ disk space: 190790 MB free
✓ database integrity: ok
✓ word search: available
! search by meaning: the model is not downloaded yet: run `agent-mem models install`
    → Run `agent-mem models install`. Word search works without it.
✓ Claude Code recording: last 2026-10-13 21:52
✓ events not saved yet: 0
! search index: last update never
    → Run `agent-mem index`.
✓ facts queue: 42 waiting, 0 done, 0 without facts, 0 failed
✓ facts model: claude (haiku), codex (gpt-6-luna), copilot (auto)
! topic links: en_core_web_sm missing; facts work without topic links
    → Run `agent-mem models install`. Set `observe.entity_model` to the language of your facts (de_core_news_sm, en_core_web_sm, ...).
✓ Claude Code hooks up to date: yes
✓ Claude Code hook command: /Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem hook claude
✓ viewer: http://localhost:37778/
✓ viewer starts at login: yes
```

`✓` passed, `!` a warning with the next step (does not fail), `✗` a failure. Exit code 0 when nothing failed. What
is checked: Python version, the command on PATH, the data folder's permissions, free disk space, database integrity,
word search, search by meaning, recording per agent, the spool and quarantine, the search index, the facts queue,
the facts model, topic links, config warnings, and per agent whether its hooks match this release and point at an
existing binary, plus the viewer and its login item.

`--fix` repairs what is safe to repair: migrates the database, rewrites hooks that an older release wrote (at the
binary path they already use), creates the data folder with private permissions, and removes leftovers of older
releases. `agent-mem upgrade` runs it for you.

## search

```text
agent-mem search "query" [--project DIR] [--all-projects] [--limit N] [--json]
```

| Option | Default | Effect |
| --- | --- | --- |
| `query` | — | words; time words such as `last week`, `gestern`, `8. April` rank that time first |
| `--project DIR` | current folder | search from this project (and its nested and surrounding projects) |
| `--all-projects` | off | every project |
| `--limit N` | 8 | number of results |
| `--json` | off | id, score, kind, time, title and candidate sources |

```json title="agent-mem search calc --json (shortened)"
[
  {"id": "M2", "score": 0.9401, "kind": "decision", "ts": "2026-10-13T20:02:55.778Z",
   "title": "calc.py must stay free of third-party imports; tests run with `uv run pytest -q`.", "sources": ["fts"]},
  {"id": "T1", "score": 0.8083, "kind": "turn", "ts": "2026-10-13T19:51:55.380Z",
   "title": "Add a subtract function to calc.py and run the tests", "sources": ["fts"]}
]
```

The CLI uses the full search (words, meaning, graph, time) like the MCP server. Results count as `shown`.

## show

```text
agent-mem show T1 M2 ...
```

```text
T1 · 2026-10-13 21:51 · claude · project repo · branch master · outcome ok · source: recorded session
request: Add a subtract function to calc.py and run the tests
actions:
  - Bash uv run pytest (failed, later fixed) — E   ImportError: cannot import name 'subtract' from 'calc' (calc.py)
  - Edit edit (ok)
  - Bash uv run pytest (ok)
result: Added `subtract` to calc.py; `uv run pytest -q` passes (2 tests).
```

Up to 10 ids, the same text `mem_get` gives an agent. Retired memories say so and name their replacement.

## pause

```text
agent-mem pause [--for 30m|2h|1d]
```

```text
$ agent-mem pause --for 2h
Recording paused until 2026-10-14 00:13.
```

Without `--for` the pause lasts until `resume` (at most 10 years with `--for`). Nothing is recorded meanwhile;
recall still works. The session panel says `Recording paused until …`.

## resume

```text
$ agent-mem resume
Recording again.
```

## export

```text
agent-mem export [--project DIR] [--output FILE]
```

```text
$ agent-mem export --output memory.json
Exported to memory.json.
```

One JSON object, `version: 2`, with `exported_at`, `projects`, `sessions`, `turns` (without events),
`memories` (including retired ones), `recipes`, `rules` and `profile`. Without `--output` it is printed. The file is
written with owner-only permissions. `--project` limits it to one project.

## purge

```text
agent-mem purge --id ID | --project DIR | --before DATE | --all   [--yes]
```

| Option | Deletes |
| --- | --- |
| `--id T12` / `--id M3` | one turn (with its events, the facts and lessons drawn from it, and the session summary) or one memory |
| `--project DIR` | everything of the project whose **root** is `DIR` (a path inside a project is refused, so a typo cannot hit the project around it) |
| `--before 2026-01-31` | turns started and memories created before the date, older events and sessions |
| `--all` | the database, spool, quarantine and logs; config and models stay |

Every option also removes the matching events from the spool and quarantine, the vectors and search keys, and
queues mem0 copies for deletion. After a partial purge the database is vacuumed. An open offline snapshot page is
discarded. Without `--yes` a terminal asks; a script is refused:

```text
$ agent-mem purge --all
Refusing to delete ALL agent-mem data without --yes.
```

## observe

```text
agent-mem observe [--drain | --once] [--status] [--retry-failed] [--reprocess-raw] [--reindex] [--quiet] [--json]
```

| Option | Effect |
| --- | --- |
| *(none)* | process what is due, wait `observe.idle_seconds` for more, then review and graph |
| `--drain` | the same; what the hooks start |
| `--once` | process what is due and exit without waiting |
| `--status` | queue counts, today's calls per provider and job, last errors |
| `--retry-failed` | turns that used up their attempts get new ones |
| `--reprocess-raw` | turns saved word for word go back to the queue (after installing a CLI) |
| `--reindex` | rebuild the mem0 index from SQLite, no model call |
| `--quiet`, `--json` | no output, or the stats as JSON |

```text
$ agent-mem observe --status
Requests: 42 waiting, 3 with facts, 0 saved word for word, 1 failed
  today claude, facts: 3 calls
  today claude, titles: 3 calls
  T17: claude exit 1: …
Try them again with `agent-mem observe --retry-failed`.

$ agent-mem observe
Done: 7 new facts from 3 batches of requests.
```

`Already running in the background.` when another observer holds the lock.

## index

```text
agent-mem index [--quiet] [--no-download] [--json]
```

Runs the indexer now: spool replay, stale turns, token accounting, native notes, embeddings, retention
([all steps](../architecture/background-processes.md#agent-mem-index)). `Search index is up to date.`, or
`Not updated: another indexer is running, try again shortly.` `--no-download` never fetches the search model.

## consolidate

```text
agent-mem consolidate [--json]
```

Runs the indexer with the daily consolidation forced. `Cleanup done.` The JSON shows each step's count
(`edges_pruned`, `global_preferences`, `superseded`, `rules_expired`, `commands_cleaned`, `fixes_dropped`,
`entities_dropped`, `anchors`, `invalidated`).

## rules

```text
agent-mem rules [list | enable | disable | delete] [R1]
```

```text
$ agent-mem rules
R1 [suggested: only warns] Use `pnpm` instead of `npm` · from 2 corrections · used 3 times · ignored 0 times · project repo
$ agent-mem rules enable R1
R1: enabled.
$ agent-mem rules
R1 [on: can block] Use `pnpm` instead of `npm` · from 2 corrections · used 4 times · ignored 0 times · project repo
```

`used` counts how often the rule matched a command, `ignored` how often you overrode it. `enable` and `disable`
reset the override count. The id is accepted with or without `R`.

## lessons

```text
$ agent-mem lessons
Suggested lines for AGENTS.md / CLAUDE.md (review before adding):

- Use pnpm instead of npm (you corrected this 2x)
- If `uv run pytest` fails with "E   ImportError: cannot import name …", check calc.py (fixed this way 2x)
- calc.py must stay free of third-party imports; tests run with `uv run pytest -q`.
```

For the current project: rules stated at least twice, error fixes that worked at least twice, and your
decisions and preferences. Nothing is written; you copy what you agree with. `Nothing repeated often enough yet.`
when there is nothing.

## models

```text
agent-mem models [status | install]
```

```text
$ agent-mem models
Search by meaning is not available: the model is not downloaded yet: run `agent-mem models install`. Search by words keeps working.
$ agent-mem models install
Model ready: sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2 (cache ~/Library/Application Support/agent-mem/models).
Topic model ready: de_core_news_sm (~/Library/Application Support/agent-mem/models/…).
```

`install` downloads the search model (about 220 MB, ONNX) and, with the `[observer]` extra, the spaCy topic model
named in `observe.entity_model` from spaCy's releases (checksums pinned for the English and German small models)
into the data folder, so reinstalling the tool keeps them.

## setup

```text
agent-mem setup all|claude|codex|copilot|opencode [--write] [--no-models]
```

Without `--write` it only prints the steps for each agent and what it detected:

```text
$ agent-mem setup claude
Claude Code
  1. /plugin marketplace add Abuarchiv/agent-mem
  2. /plugin install agent-mem@agent-mem
  3. Restart Claude Code. Hooks and the MCP server are registered by the plugin.
  …
Or let agent-mem write the files with absolute paths: `agent-mem setup claude --write`.

Detected:
  agent-mem binary: /Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem
  agent claude    found /Users/you/.local/bin/claude
  agent codex     found /opt/homebrew/bin/codex
  agent copilot   found /opt/homebrew/bin/copilot
  agent opencode  not installed
  facts by claude   found, uses your existing login
  facts by codex    found, uses your existing login
  facts by copilot  found, uses your existing login
  facts by ollama   not installed (optional)
  search by meaning: installed
  facts: installed
```

With `--write`, for each agent (`all` means every agent on PATH):

1. writes its hooks and MCP entry with the absolute path of the binary ([which files](../architecture/hooks.md#how-every-hook-is-launched));
2. says which CLI will write facts for it and with which model;
3. downloads the search and topic models (skip with `--no-models`);
4. registers and starts the viewer login item;
5. tells Codex users to trust the hooks once in `/hooks`, and Claude Code users to keep only one of plugin and
   `setup --write` if both are on.

## uninstall

```text
agent-mem uninstall all|claude|codex|copilot|opencode
```

```text
claude: removed agent-mem hooks and MCP entries from:
  /Users/you/.claude/settings.json
Data stays in ~/Library/Application Support/agent-mem; `agent-mem purge --all` deletes it.
```

Removes exactly what `setup --write` wrote; your own hooks stay. When no agent is left, the viewer's login item
goes too.

## view

```text
agent-mem view [--project DIR] [--output FILE|FOLDER] [--no-open]
```

Opens `http://localhost:37778/` (`?project=<id>` with `--project`), starting the viewer first if needed. If it
cannot start within 8 seconds, or with `--output` or `--no-open`, it writes an offline snapshot instead:

```text
Wrote agent-mem.html. It is a local file with a copy of your memory; it makes no network requests.
```

A folder as `--output` gets `agent-mem.html` inside it.

## serve

```text
agent-mem serve [--quiet] [--autostart | --no-autostart]
```

Runs the viewer in the foreground (`Viewer at http://localhost:37778/ (open it with agent-mem view). Stop with
Ctrl+C.`). `--autostart` registers the login item and starts it; `--no-autostart` removes it. See
[Background Processes](../architecture/background-processes.md#agent-mem-serve).

## upgrade

```text
agent-mem upgrade [--extras semantic observer]
```

```text
$ uv tool install --upgrade "agent-mem[semantic,observer] @ git+https://github.com/Abuarchiv/agent-mem"
agent-mem 1.0.0 (7d6cf39) -> 1.0.0 (86ad40c). Hooks use it now; running sessions keep their MCP server until restart.
$ agent-mem models install
$ agent-mem doctor --fix
```

Installs the newest commit of the source you installed from (a branch or tag you named stays), keeps your extras and
adds the ones you name, then runs the new code's `models install` and `doctor --fix`. Only for `uv tool` installs;
otherwise it says how to upgrade. Never use `uv tool install --reinstall` for this: it deletes the environment
first, and hooks firing meanwhile find no binary.

## paths

```text
$ agent-mem paths
Data folder:  ~/Library/Application Support/agent-mem
Database:     ~/Library/Application Support/agent-mem/memory.db
Settings:     ~/Library/Application Support/agent-mem/config.json
Logs:         ~/Library/Application Support/agent-mem/logs
Models:       ~/Library/Application Support/agent-mem/models
```

`--json` gives `data_dir`, `database`, `config`, `logs`, `models`.

## mcp

```text
agent-mem mcp
```

The stdio MCP server. Your agent starts it from its MCP configuration; you never need to run it yourself. See
[the five tools](../architecture/search.md#the-five-mcp-tools).

## eval

```text
agent-mem eval DATASET [--limit N] [--subset FILE] [--progress FILE] [--no-semantic]
               [--backend sqlite|mem0] [--llm none|ollama|claude|codex|copilot] [--llm-model M]
               [--custom-prompt FILE] [--embedder MODEL] [--vector-floor X]
agent-mem eval --fixture tests/fixtures/memory_cases.json [--min-score X]
```

For developers: runs LongMemEval-S (or the repository's fixture) against a fresh database and prints the report as
JSON. `--backend mem0` compares with mem0's own search; `--llm` lets mem0 extract with that CLI. `--progress`
writes one line per finished question and resumes from it. See [Measurements](../measurements.md).

## hook

```text
agent-mem hook claude|codex|copilot|opencode [event]
```

Called by your agent with the event JSON on stdin; it always exits 0. Typed by a person without stdin it only
explains itself. See [How agent-mem uses hooks](../architecture/hooks.md).

> [!NOTE]
> `hook` is not in `agent-mem --help`'s list: it has its own fast path that imports only the standard library, before
> the argument parser is loaded.

---

← [.agent-mem.json](project-settings.md) · [Contents](../../README.md#contents) · [Environment variables](environment.md) →
