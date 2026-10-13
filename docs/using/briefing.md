# Session briefing

What the agent gets at the start of every session, what you see, and how to make it bigger or smaller.

Every new session starts with a briefing: what this project's memory holds that the agent should know before it
does anything. Subagents get one too. After compaction a shorter block takes its place.

## What is in it

| Section | Example | From |
| --- | --- | --- |
| Current time | `Current: 2026-10-13 21:52 CEST` | so the agent can tell how old each entry is |
| Problems to act on | `! agent-mem: codex hooks failing (…). Run agent-mem doctor.` | only when something is broken |
| Commands that worked | `test: uv run pytest -q; lint: uv run ruff check src` | successful test, build, lint and install commands |
| Preferences and rules | `Use pnpm instead of npm (corrected 2x)` | your corrections |
| Decisions, failed attempts, facts | `[M2 · decision] calc.py must stay free of third-party imports` | the observer, your statements, reverts |
| Lessons | `When a new function is added to calc.py, export it in __all__ …` | mistakes and their fixes |
| The other agent's notes | a Claude Code note, shown to Codex | notes the agents keep themselves |
| Focus | memories that match the last session's open steps | a search with "next steps" as the query |
| Still open | `[unresolved] uv run pytest failed: …` | a test or build that kept failing in the last 72 hours |
| Last session | its title, what was done, what is next | the observer's session summary |
| Recent work | `[T41 · work, bugfix] Fixed missing subtract import in calc.py` | titles of recent requests |

Entries are chosen by **importance and use, not by age**. A preference you stated twice ranks above a fact nobody
opened in weeks, and a memory shown at many session starts that the agent never used makes room for something
else. The chosen entries are then printed oldest first, so a later entry reads as an update of an earlier one. The
agent is told that newer wins when two disagree.

## What you see

Claude Code, Codex and Copilot show you the same entries as a panel, grouped by kind, plus the project's counts,
which model writes facts, and where to look further. See [the terminal panel](../architecture/context-injection.md#the-terminal-panel).

## Size

The briefing fills up to `budgets.session_start`, **8,000 tokens** by default. A new project gets a short block, a
busy one gets close to the budget. Claude Code receives at most 2,400 tokens per block and Codex 7,800, so in
practice those limits apply there.

```json title="config.json"
{ "budgets": { "session_start": 2000 } }
```

About a quarter of the default. `0` turns the briefing off; the panel then only shows the counts.

## Opening an entry

Each line starts with an id. The agent opens it with `mem_get` (`T42` a request in full, `M7` a memory with its
provenance), or you do with `agent-mem show T42`. `(+48 tok)` after a line says opening it would cost about 48
more tokens.

> [!NOTE]
> What the briefing never contains: memories retired by a newer one, memories from this session's own requests, and
> the agent's own notes (it already loads them itself).

---

← [OpenCode](../agents/opencode.md) · [Contents](../../README.md#contents) · [Reminders during work](recall-during-work.md) →
