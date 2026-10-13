# Architecture Overview

The parts of agent-mem, how data flows through them, and where each piece lives in the code.

agent-mem has three jobs. It **records** what coding agents do through their hooks, it **thinks** about it later
in background processes that call the agent's own CLI, and it **recalls** what matters, injected into the next
session, searchable over MCP and visible in a local viewer. One SQLite file holds all of it.

![Agents call the hook, which writes memory.db (or the spool) and returns context; observe and index work in the background with the agents' CLIs and mem0; the MCP server, the viewer and the CLI read memory.db.](../../assets/diagrams/system-overview.svg)

<sub>[Excalidraw scene](../../assets/diagrams/system-overview.excalidraw)</sub>

## Components

| Component | Command | Runs | Role |
| --- | --- | --- | --- |
| Hooks | `agent-mem hook <harness> [event]` | per event, ~65 ms | record, inject, warn, block; start the rest |
| Observer | `agent-mem observe` | after requests, until idle | facts, titles, session summary, lessons, review, graph |
| Indexer | `agent-mem index` | after requests, until done | spool replay, embeddings, notes, token accounting, consolidation, retention |
| MCP server | `agent-mem mcp` | per agent session | `mem_search`, `mem_timeline`, `mem_get`, `mem_remember`, `mem_forget` |
| Viewer | `agent-mem serve` | always on, port 37778 | a read-only page over the database |
| CLI | `agent-mem …` | on demand | setup, status, doctor, search, purge, export, … |

## Technology

| Layer | Choice | Why |
| --- | --- | --- |
| Language | Python 3.11–3.14 | the hook path uses the standard library only |
| Storage | SQLite (WAL), FTS5 with the trigram tokenizer | one file, concurrent readers, substring search in any language |
| Vectors | fastembed (ONNX Runtime), `paraphrase-multilingual-MiniLM-L12-v2`, numpy | multilingual, no PyTorch, about 220 MB |
| Fact extraction | mem0 with the agent's CLI as its model (`claude -p`, `codex exec`, `copilot -s`), optional Ollama | no API key; the vendor that saw a request writes its facts |
| Fact index | Qdrant in local path mode (through mem0) | mem0's deduplication; rebuildable from SQLite |
| Topic links | spaCy (`en_core_web_sm` or another language) | mem0's entity store |
| Time words | dateparser | "letzte Woche", "8. April", "last friday" |
| MCP | the official Python MCP SDK, stdio | one process per session |
| Viewer | Python `http.server`, a self-contained HTML page with its own script | no build step, no network |
| Installer | `uv tool` | one isolated environment, in-place upgrades |

## Data flow

### Recording

1. The agent fires an event and runs `agent-mem hook <harness>` with the payload on stdin.
2. The payload is normalized to one `Event`, checked (internal, disabled, paused, excluded), cleaned of private
   sections and secrets, and written in one transaction: session, event, turn, recipes, rules, dead ends.
3. If the database is busy or the 400 ms deadline passes, the event goes to `spool/` instead.
4. A closed turn is queued in `observe_queue`.

### Thinking

1. `Stop` starts `agent-mem observe --drain`. It leases up to five turns of a session and builds a bundle: prompts,
   what the tools showed, answers, subagent answers.
2. The agent's own CLI extracts facts through mem0; agent-mem mirrors them into `memories`.
3. A second call writes turn titles, the running session summary and lessons.
4. Once a day per project the facts are reviewed for duplicates and outdated ones; new memories are read into a
   graph of things and relations.
5. `agent-mem index` replays the spool, embeds new text, reads the agents' own notes, measures token work and runs
   the daily consolidation.

### Recalling

1. **Session start:** a briefing ranked by importance and use, within 8,000 tokens.
2. **Each prompt:** confident matches only, within 2,400 tokens.
3. **Before a tool:** rule notes or a block, and the history of the files it touches.
4. **After a failure:** how this error was fixed before.
5. **After compaction:** session goal, decisions, open failures, files.
6. **On demand:** the MCP tools, `agent-mem search`, the viewer.

## Session lifecycle

| Moment | Event | What agent-mem does |
| --- | --- | --- |
| open | `SessionStart` | briefing, viewer started, panel |
| ask | `UserPromptSubmit` | close the previous turn, open a new one, learn rules, recall |
| work | `PreToolUse` / `PostToolUse` | warn or block, file history, record, fix hints, recipes, dead ends |
| done | `Stop` | answer stored, turn closed and queued, observer and indexer started |
| compact | `PreCompact` / `SessionStart(compact)` | the compaction block |
| close | `SessionEnd` | last turn closed, summary line |

## Code layout

```text
src/agent_mem/
  core/      config, db + schema/, events, identity, privacy, signals, spool, timeutil, logutil, paths
             standard library only; shared by every path
  hooks/     hook (entry point), normalize/ (claude, codex, copilot, opencode), capture, inject
             the hook path: payload in, SQLite write, context out; standard library only
  memory/    store, search, semantic, learn, consolidate, views, indexer, notes, work
             memories, retrieval, embeddings, learning, the background indexer
  observe/   queue, bundle, providers, mem0_adapter, summary, review, relations, worker,
             spacy_model, coding_instructions.md   — the only place that imports mem0 and Qdrant
  mcp/       server: the stdio MCP server
  cli/       argument parsing, commands, install (setup --write, uninstall), health (status, doctor)
  ui/        the self-contained page and the loopback server
  bench/     evaluate: fixture and LongMemEval-S runner (not used at runtime)
plugins/     claude, codex, copilot, opencode: hook files, manifests, the OpenCode plugin
```

Dependencies point one way: `core` ← `hooks`, `memory` ← `observe`, `mcp`, `ui`, `bench` ← `cli`. Nothing in
`core`, `hooks` or `memory` imports from `observe`, `cli` or `ui`. A test checks that the hook path never imports
mem0, Qdrant, fastembed or numpy.

## Guarantees

- **The agent is never blocked or broken.** Every hook exits 0 and prints valid output or nothing.
- **No event is lost.** Busy database, migration or deadline: the event waits in the spool and is replayed in order.
- **No turn is lost.** It stays in `observe_queue` until facts are extracted or it is stored word for word; failures
  back off and end in `failed`, retryable.
- **A request stays with its vendor.** A Claude Code turn goes to Claude, a Codex turn to Codex, a Copilot turn to
  Copilot, unless you set `observe.cross_vendor`.
- **SQLite is the truth.** Everything in `mem0/` can be rebuilt from it without a model call.
- **Nothing leaves the machine** except model downloads and your agent's own CLI talking to its own provider.

## Deeper

- [How agent-mem uses hooks](hooks.md): Why hooks, how they are launched, each event, timing, failure modes, debugging.
- [Hook Lifecycle](hook-lifecycle.md): Payloads and outputs per agent, sequence diagrams, state machines.
- [Observer Pipeline](observer-pipeline.md): Queue, bundles, providers, extraction, work log, review, graph.
- [Database](database.md): Every table, the FTS5 index, migrations, retention.
- [Search Architecture](search.md): Candidates, fusion, scoring, abstention, the five MCP tools.
- [Context Injection](context-injection.md): Every block, its budget and how it is chosen.

---

← [Environment variables](../configuration/environment.md) · [Contents](../../README.md#contents) · [How agent-mem uses hooks](hooks.md) →
