# Reminders during work

What agent-mem tells the agent while it works: matching memories, rule notes, file history, fix hints.

After the briefing, agent-mem speaks only when it has something specific to say. You see none of it in the terminal.
The agent gets it as context, framed as data.

## When you send a request

agent-mem searches memory with your prompt and gives the agent the matches it is **confident** about:

```text
- [T1 · 2026-10-13 · claude] Add a subtract function to calc.py and run the tests → Added `subtract` to calc.py; `uv run pytest -q` passes (2 tests). (files: calc.py)
```

Confident means: a high enough score, and at least two of the question's content words (one if it has only one) in
the match, covering at least half of them. A match by meaning must still share a third of the words. No confident
match, no context: a weak match cannot be told from an unrelated one, and noise is not context.

Time words help. "What did we do last week about the parser?" ranks last week's requests first.

## Before a command or an edit

**A rule you taught it.** After you said "use pnpm instead of npm" twice:

```text
- Note (R1): earlier correction — Use `pnpm` instead of `npm` (2x).
```

Switch the rule on (`agent-mem rules enable R1`) and `npm` is **blocked** with that reason. See
[Rules & Lessons](rules-and-lessons.md).

**The history of a file.** Before the agent reads or edits a file that memories are linked to:

```text
Earlier memories about this file (history, not its current contents):
- Caution [M12 · 2026-10-09] calc.py: Changes to calc.py were reverted (`git checkout`) — context: Make subtract accept strings
- Decision [M7 · 2026-10-12] calc.py: Keep calc.py free of third-party imports
```

Failed attempts first, then decisions and lessons, then facts. Up to three per file, each memory once per session.
Facts older than the file's last change are left out because they may describe code that is gone. A small file
(under 1.5 KB) gets no history when it is only read; reading it costs about as much.

## When a command fails

If the same error was fixed before in this project:

```text
- This error was seen before with `uv run pytest` and fixed by changing calc.py (see T1). Seen 1x, fixed 1x.
```

Only for tests, builds, lints and installs. A failed `cat` or `grep` is looking around, not an error worth a recipe.

## After compaction

When the agent's context is compacted, it gets back what the cut part must not lose:

```text
- Session goal (T41): Add a subtract function to calc.py and run the tests
- [M7 · decision] Keep calc.py free of third-party imports
- Still failing: `uv run pytest` E   ImportError: …
- Files touched: calc.py, test_calc.py
```

## Sizes

| Reminder | Budget (tokens) | Key |
| --- | --- | --- |
| matching memories | 2,400 | `budgets.prompt` |
| rule note | 300 | `budgets.warning` |
| file history | 1,200 | `budgets.file_history` |
| fix hint | 400 | `budgets.failure` |
| after compaction | 4,000 | `budgets.compact` |

`0` turns one off. To get more or fewer matches per prompt, change `retrieval.inject_min_score` (0.35) and
`retrieval.min_term_coverage` (0.5); lower means more context and more noise.

> [!TIP]
> Agents can also look things up themselves with the MCP tools: `mem_search`, `mem_timeline` and `mem_get`. See
> [Search & MCP](search-and-mcp.md).

---

← [Session briefing](briefing.md) · [Contents](../../README.md#contents) · [Search & MCP tools](search-and-mcp.md) →
