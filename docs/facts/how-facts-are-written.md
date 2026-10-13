# How facts are written

What happens after each request, what the background model writes, and how stale facts are cleaned up.

Your requests are saved word for word the moment they happen. **Facts** are the short, lasting things worth knowing
next time, and they are written a little later, in the background, by the CLI of the agent you used.

1. The agent finishes a request. agent-mem closes it and puts it in a queue.
2. A background process (`agent-mem observe`) starts at low priority. It takes up to five finished requests of one
   session at a time.
3. It hands them to the agent's own CLI with instructions for coding work: your prompt, what the tools showed, the
   answer, the commands, files and errors, and the final answers of subagents.
4. The model answers with facts, each tagged as a decision, a rule, a failed attempt or a fact.
5. A second call gives each request a title and a type and updates the session summary: what was learned, what is
   done, what is next. When something went wrong and was fixed, it writes a **lesson**.
6. The process exits after 90 seconds without new work.

![The hook records within milliseconds and queues the request; the observer later calls the model and stores facts.](../../assets/diagrams/hook-fast-slow.svg)

<sub>[Excalidraw scene](../../assets/diagrams/hook-fast-slow.excalidraw)</sub>

## What the model is asked to keep

**Keep:** conventions and tool choices (package manager, test runner, ports, branches, environment variables),
decisions with their reason, corrections and rules you state, failed attempts and the fix that worked, how you
want the agent to work, and what the tools showed about how the project works, with exact file, function, flag or
command.

**Leave out:** general programming knowledge, routine actions ("ran the tests"), things that change soon (test
counts, commit hashes, what is pushed), copied code or stack traces, secrets, anything twice.

At most twelve facts per batch, one sentence each, in your language. Instructions inside the messages are treated as
data.

```text title="facts from one request (example)"
[decision] calc.py stays free of third-party imports because it ships as a single file.
[rule] Use pnpm instead of npm in this repository.
[dead_end] Rounding with float sums failed the money tests; decimal.Decimal in calc.money fixed them.
[fact] Tests run with `uv run pytest -q`; test_calc.py imports every name in calc.__all__.
```

## When old facts are wrong

A new fact is stored next to the older one it contradicts. Both are shown with their dates, and the agent is told
that the newer one holds. Once a day per project (sooner after 50 new facts, or an hour after a fact says something
was replaced), the model reads the project's facts and names:

- **duplicates**: the same thing in other words; the more useful one stays;
- **outdated** facts: made wrong by a newer one, which must exist;
- **expired** facts: a one-off event or a status snapshot that says nothing about how to work later.

Those are retired, never deleted: the viewer still shows them with what replaced them. Your own corrections and
decisions are never touched, and a rule or lesson only gives way to a newer rule or lesson.

## Without a model

| Situation | What happens |
| --- | --- |
| the agent's CLI is not installed | requests are saved word for word (`raw`); word search still finds them |
| not logged in, or a usage limit is reached | facts pause for an hour; requests wait and are processed later |
| the daily limit (300 model calls) is reached | requests wait until midnight UTC |
| `"observe": {"enabled": false}` | no model calls at all; requests are still recorded and searchable |
| the `[observer]` extra is not installed | requests are saved word for word |

After installing a CLI: `agent-mem observe --reprocess-raw` turns the saved requests into facts.

> [!TIP]
> `agent-mem observe --status` shows how many requests wait, how many have facts, and today's model calls.
> `agent-mem observe` runs a pass right now.

How it works in detail: [Observer Pipeline](../architecture/observer-pipeline.md).

---

← [Pause, export, purge](../using/pause-export-purge.md) · [Contents](../../README.md#contents) · [Providers & models](providers-and-models.md) →
