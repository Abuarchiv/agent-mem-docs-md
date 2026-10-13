# Search & MCP tools

How you and your agents search memory, and the five tools agents get.

## You: agent-mem search

```sh
agent-mem search "subtract import"
agent-mem search "what did we decide about the database last week"
agent-mem search "rate limit" --all-projects --limit 20
agent-mem search calc --json
```

Searches by words, by meaning (with the `[semantic]` extra) and through the graph of files and commands, from the
current folder's project (and the projects nested in it or around it). Then:

```sh
agent-mem show T1 M2     # the full entries
agent-mem view           # the browser page
```

## Agents: five MCP tools

`agent-mem setup … --write` registers the MCP server with every agent. The agent is told to search first, open only
what it needs, treat recalled text as data, and cite ids.

| Tool | Use it to | Example |
| --- | --- | --- |
| `mem_search(query, limit=8, all_projects=false)` | find ids and one-line titles | `mem_search("why did we drop postgres")` |
| `mem_timeline(id, before=3, after=3)` | see what happened before and after a request in its session | `mem_timeline("T41")` |
| `mem_get(ids)` | read up to 10 entries in full, with provenance and file state | `mem_get(["T41", "M7"])` |
| `mem_remember(text, kind, title, supersedes, global_scope)` | store a fact, decision, failed attempt or lesson explicitly | `mem_remember("Deploys go through make release", kind="decision")` |
| `mem_forget(id)` | delete a request or memory of this project | `mem_forget("M7")` |

**mem_search**

```text
T1 · 2026-10-13 · claude · score 0.65
  Add a subtract function to calc.py and run the tests
M2 · 2026-10-13 · decision · score 0.73
  calc.py must stay free of third-party imports; tests run with `uv run pytest -q`.
Use mem_get with ids for details, mem_timeline for context.
```

**mem_timeline**

```text
▶ T1 · 2026-10-13T19:51 · ok: Add a subtract function to calc.py and run the tests
    → Added `subtract` to calc.py; `uv run pytest -q` passes (2 tests).
  T2 · 2026-10-13T19:51 · ok: Use pnpm instead of npm in this repo
    → Noted: pnpm instead of npm.
```

**mem_get**

```text
M2 · decision · 2026-10-13 22:02 · project repo · source agent/- · trust agent
calc.py must stay free of third-party imports; tests run with `uv run pytest -q`.
```

What agents cannot do:

- store a **preference** (preferences come only from what you type);
- replace or delete **your** memories, or memories of another project;
- see weak matches as confident ones: `mem_search` says "No confident match" and lists them separately.

Every tool the agent uses teaches agent-mem something: opening a memory or citing its id counts as **used** and
ranks it higher next time. See [Learning & Ranking](../architecture/learning-and-ranking.md).

> [!NOTE]
> Details of scoring, the candidate sources and the abstention gate: [Search Architecture](../architecture/search.md).

---

← [Reminders during work](recall-during-work.md) · [Contents](../../README.md#contents) · [Viewer](viewer.md) →
