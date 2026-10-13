# Copilot CLI & app

How agent-mem connects to GitHub Copilot CLI and the Copilot app, and the Copilot-specific details.

## Connect

```sh
agent-mem setup copilot --write
```

Writes `~/.copilot/hooks/agent-mem.json` (a file of its own) and adds the MCP server to
`~/.copilot/mcp-config.json`:

```json
"agent-mem": { "type": "local", "command": "/Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem", "args": ["mcp"], "tools": ["*"] }
```

Copilot CLI and the Copilot app (macOS) both read hooks from `~/.copilot/hooks`. `COPILOT_HOME` moves `~/.copilot`.

> [!WARNING]
> Use `setup --write`, not a Copilot plugin. Copilot does not fire `preToolUse` hooks that a plugin defines
> ([copilot-cli#2540](https://github.com/github/copilot-cli/issues/2540)); without them there are no rule notes,
> blocks or file history before a tool.

## Hooks

Copilot passes the event name as an argument (`agent-mem hook copilot sessionStart`) and has a `bash` and a
`powershell` form of each command, both with `timeoutSec: 10`:

| Event | agent-mem does |
| --- | --- |
| `sessionStart` | briefing, progress lines |
| `subagentStart` | briefing for the subagent |
| `userPromptSubmitted` | new turn, rules, confident matches |
| `preToolUse` | rule note or deny, file history |
| `postToolUse` | record, recipes, fix hints, file history (when nothing else was said) |
| `postToolUseFailure` | the failure, fix hint |
| `errorOccurred` | recorded as a failed tool named `error` |
| `agentStop` | the answer, read from the transcript; close the turn |
| `subagentStop` | the subagent's answer |
| `preCompact` | recorded |
| `sessionEnd` | close, a short summary |

## What Copilot does differently

- **No `systemMessage`.** agent-mem writes a `{"type": "progress", "message": "…"}` line before the JSON; Copilot
  shows it in its timeline and parses the rest. The session panel appears as progress lines, uncoloured.
- **No answer in `agentStop`.** Only the transcript path. agent-mem reads the end of
  `session-state/<id>/events.jsonl` and waits up to 250 ms for the final message, which Copilot writes 50 to 150 ms
  after the hook starts.
- **The first prompt arrives before `sessionStart`.** Memories from this session's own turns are kept out of its
  briefing.
- **`toolArgs`** is an object since CLI 1.0.83 and a JSON string before; both are read.
- **Hook output limits.** `postToolUse` takes 10 KB, `sessionStart` 10 MiB; agent-mem caps blocks around tools at
  2,400 tokens.

## The Copilot app

Sessions the app runs in its worktrees (`~/.copilot/copilot-worktrees/<repo>`) are recorded for their repository.
Plain chats in the app, without a repository, are not recorded.

## Facts

```bash
COPILOT_HOME=<private temp> copilot -s --no-custom-instructions --disable-builtin-mcps --model auto --auto-tier efficiency
```

A private `COPILOT_HOME` per call, so the call is not saved as a session in the app and runs none of your hooks,
plugins or MCP servers; `config.json` is copied in for logins kept there. `auto` with the efficiency tier works on
every plan, including Free, which refuses fixed models; it resolves to GPT-6-Luna.

## Token accounting

From `~/.copilot/session-state/<id>/events.jsonl`: each `session.usage_record`'s input minus cache reads, plus output.

Tested with Copilot CLI 1.0.96 and the Copilot app 1.1.29 on macOS.

---

← [Codex](codex.md) · [Contents](../../README.md#contents) · [OpenCode](opencode.md) →
