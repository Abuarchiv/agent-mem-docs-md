# Rules & lessons

How corrections become rules that warn or block, and how mistakes become lessons for next time.

## Rules from your corrections
1. You type a correction, for example "use pnpm instead of npm". agent-mem records a **suggested rule** R1: avoid
   `npm`, prefer `pnpm`.
2. Before the next command that runs `npm`, every agent gets a note:
   `Note (R1): earlier correction — Use pnpm instead of npm (1x).`
3. The second time you say it (`rules.min_corrections`, 2), it also becomes a **preference** that every briefing
   carries: `Use pnpm instead of npm (corrected 2x)`.
4. Switch it on to block the command:

   ```sh
   agent-mem rules enable R1
   ```

   Now `npm install` is denied before it runs, and the agent reads why:
   `agent-mem rule R1: Use pnpm instead of npm (you were corrected 2x).`
Recognized forms (English and German): `use X instead of Y`, `nutze / verwende / nimm X statt Y`, `X statt Y`,
`nicht X benutzen, sondern Y`, `never use X`, `nie X`, `kein X`. A rule the observer extracts counts only if both
tool names appear in what you typed, not just in a file the agent read.

### Managing rules

```text
$ agent-mem rules
R1 [suggested: only warns] Use `pnpm` instead of `npm` · from 2 corrections · used 3 times · ignored 0 times · project calc
```

| Command | Effect |
| --- | --- |
| `agent-mem rules enable R1` | the rule blocks |
| `agent-mem rules disable R1` | back to a note |
| `agent-mem rules delete R1` | gone |
| `"rules": {"auto_enable": true}` | rules switch on by themselves after `min_corrections` |

A rule turns itself off when:

- you override it twice: typing `anyway`, `trotzdem`, `egal`, `ignore the rule` or `do it anyway` within 15 minutes
  after it fired;
- you say the opposite: "use npm instead of pnpm" disables the old rule and retires its preference;
- it has not fired for `rules.expire_days` (90).

A preference you state in **two projects** becomes a global preference for every project.

## Lessons from mistakes

When a request shows a mistake and what fixed it, a correction by you, or an approach that was reverted, the
background model writes a **lesson**, a rule for next time:

```text
When a new function is added to calc.py, export it in __all__ before running the tests, because the import test reads __all__.
```

At most two per batch of requests, usually none, and only with a request as evidence. Lessons get their own slots in
the briefing and are ranked by use like everything else.

## Error fixes and failed attempts

Two more things are learned without any model:

- **Error fixes.** A test, build, lint or install fails, then passes after edits: the next time that error appears,
  the agent is told which files fixed it.
- **Failed attempts.** Changes reverted with `git checkout -- file`, `git restore`, `git revert` or
  `git reset --hard` become a "Caution" in the history of those files.

## Into AGENTS.md or CLAUDE.md

When something keeps coming up, it may belong in your repository's instructions:

```text
$ agent-mem lessons
Suggested lines for AGENTS.md / CLAUDE.md (review before adding):

- Use pnpm instead of npm (you corrected this 2x)
- If `uv run pytest` fails with "E   ImportError: cannot import name …", check calc.py (fixed this way 2x)
- calc.py must stay free of third-party imports; tests run with `uv run pytest -q`.
```

agent-mem never writes these files. Copy what you agree with.

> [!NOTE]
> The viewer's **Learned** page lists lessons, error fixes, rules and commands, each linked to the request it came
> from.

---

← [Viewer](viewer.md) · [Contents](../../README.md#contents) · [Privacy](privacy.md) →
