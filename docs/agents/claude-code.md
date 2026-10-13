# Claude Code

How agent-mem connects to Claude Code, what it shows there, and what to know.

## Connect

**setup --write (recommended)**

```sh
agent-mem setup claude --write
```

Writes the hooks into `~/.claude/settings.json` (merged with yours) and registers the MCP server with
`claude mcp add --scope user agent-mem -- <path>/agent-mem mcp`. If `claude` is not on `PATH`, setup prints that line
for you to run.

**Plugin**

```text
/plugin marketplace add Abuarchiv/agent-mem
/plugin install agent-mem@agent-mem
```

The plugin registers hooks and the MCP server but not the command itself: install it first with
`uv tool install …`. Use the plugin **or** `setup --write`, not both, or every hook runs twice; `setup --write`
warns when both are on.

`CLAUDE_CONFIG_DIR` moves `~/.claude`, and setup follows it.

## Hooks

| Event | Matcher | Mode | Timeout | agent-mem does |
| --- | --- | --- | --- | --- |
| `SessionStart` | — | sync | 10 s | briefing, panel, viewer start; after compaction the compact block |
| `SubagentStart` | — | sync | 10 s | the briefing for each subagent |
| `UserPromptSubmit` | — | sync | 10 s | new turn, rules, confident matches |
| `PreToolUse` | `Bash\|Edit\|Write\|MultiEdit\|NotebookEdit` | sync | 10 s | rule note or deny, file history |
| `PostToolUse` | `*` | **async** | 10 s | record, recipes, dead ends, feedback |
| `PostToolUseFailure` | `*` | sync | 10 s | record the failure, fix hint |
| `Stop` | — | **async** | 10 s | answer, close the turn, start the observer |
| `SubagentStop` | — | **async** | 10 s | the subagent's answer for the bundle |
| `PreCompact` | — | **async** | 10 s | recorded |
| `SessionEnd` | — | **async** | 10 s | close, summary line |

Payloads and outputs are on [Hook Lifecycle](../architecture/hook-lifecycle.md).

## What you see

- **Session start and after compaction:** the panel, in colour (`NO_COLOR` turns colour off), via `systemMessage`.
- **Session end:** `agent-mem session saved · 2 requests · 1 file · facts are added in the background`.
- **Failures** you must act on, in red, at most every ten minutes.
- Nothing for ordinary prompts and tool calls.

The model gets context through `hookSpecificOutput.additionalContext` on session start, prompts and around tools,
and a deny through `permissionDecision: "deny"`.

## Limits

Claude Code passes at most 10,000 characters per hook string to the model and moves the rest to a file the model is
not told to read. agent-mem caps every block for Claude Code at 2,400 tokens, whatever the budget says.

## Facts

Claude Code requests are turned into facts by `claude -p … --model haiku` under your login, in an empty temporary
folder, with no tools, no MCP servers, no settings and no saved session. `observe.claude_model` changes the model;
a model your plan does not offer is retried with the CLI's default. A usage limit or a missing login pauses facts for
an hour; requests wait.

## Claude Code's own memory

Claude Code keeps notes in `~/.claude/projects/<folder>/memory/*.md` and loads them itself. agent-mem reads them as
`native_note` memories and shows them to **Codex, Copilot and OpenCode**, never back to Claude Code. A note deleted
in Claude Code is retired in agent-mem too. `"native_notes": false` turns this off.

## Token accounting

Claude Code's transcripts (`~/.claude/projects/<folder>/<session>.jsonl` and subagent transcripts) give the work
behind each request: input, cache writes and output. See [Token Accounting](../architecture/token-accounting.md).

> [!TIP]
> `/hooks` in Claude Code shows the registered hooks; `claude --debug` prints each hook call and its output.
> `agent-mem doctor` says whether the hooks match this release.

---

← [Upgrade & uninstall](../getting-started/upgrade-uninstall.md) · [Contents](../../README.md#contents) · [Codex](codex.md) →
