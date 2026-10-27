# OpenCode

The agent-mem plugin for OpenCode, what it maps, and what OpenCode does not get.

## Connect

```sh
agent-mem setup opencode --write
```

Copies the plugin to `~/.config/opencode/plugins/agent-mem.ts` (with the absolute path of the command) and adds the
MCP server to `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": { "agent-mem": { "type": "local", "command": ["/Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem", "mcp"], "enabled": true } }
}
```

`XDG_CONFIG_HOME` moves `~/.config`.

## The plugin

OpenCode has no command hooks; it loads TypeScript plugins. agent-mem's one file serves both OpenCode lines: OpenCode
2 calls its `setup` and streams events, OpenCode 1.x (the npm package) loads its default export and calls `server()`.
Each event goes to `agent-mem hook opencode <event>` as JSON on stdin; the call is killed after 10 seconds, the same
limit as the other agents' hooks (on Windows one call took 2.3 to 3.2 s under virus scanning). Calls run one after
another, so a session's row exists before its first prompt is looked up.

### OpenCode 2 (event stream)

| Event | Sends | agent-mem does |
| --- | --- | --- |
| first event of a session | `session.start` | new session |
| `session.inbox.enqueued` (a user item) | `chat.message` with the prompt | new turn |
| `session.tool.success` / `session.tool.failed` | `tool.after` with name, input, output and exit code (0 or 1) | record, recipes |
| `session.text.ended` | — | adds to the turn's answer |
| `session.execution.succeeded` / `failed` / `interrupted` | `session.idle` with the answer | close the turn, start the observer |

OpenCode 2's plugin API has no hook before a tool and none that changes the system prompt, so there agent-mem
**records only**: no briefing, no matches, no rule notes or deny. The search is still there through the MCP server.
Each loaded location gets its own plugin instance; events of another location are skipped so a session is recorded
once.

### OpenCode 1.x (hooks)

| OpenCode hook | Sends | agent-mem does |
| --- | --- | --- |
| `event` `session.created` | `session.start` | briefing, held for the next model call |
| `chat.message` | `chat.message` with the prompt's text parts (not synthetic ones) | new turn, rules, matches |
| `tool.execute.before` | `tool.before` | rule note or **deny** (the plugin throws, which stops the tool), file history |
| `tool.execute.after` | `tool.after` with output and exit code | record, recipes, fix hints |
| `experimental.text.complete` | — | remembers the turn's last text |
| `event` `session.idle` | `session.idle` with that text | close the turn, start the observer |
| `experimental.session.compacting` | `session.compacting` | the compact block goes into the compaction prompt |
| `experimental.chat.system.transform` | — | adds held context to the system prompt of the next call |

Any error resolves to "no context"; the only error the plugin throws on purpose is an enabled rule. Calls to
agent-mem's own tools are skipped.

## Facts

OpenCode requests get facts from `opencode run --standalone --agent plan --model opencode/big-pickle`, a free model
(`observe.opencode_model`). `--standalone` keeps the run off the shared OpenCode server, `--agent plan` keeps it
read-only, and agent-mem's own plugin skips it. See [Providers and models](../facts/providers-and-models.md).

## What OpenCode does not get

- **No panel.** OpenCode has no channel for a message to you.
- **No context in OpenCode 2** (see above); OpenCode 1.x gets it.
- **No token accounting.** OpenCode's work per request is not measured.

> [!WARNING]
> Run OpenCode sessions one at a time per repository. Two at once can collide on OpenCode's own snapshot lock; this is
> an OpenCode limitation, not agent-mem's.

Tested with OpenCode 1.19 (npm, install script) and 2.2.

---

← [Copilot CLI & app](copilot.md) · [Contents](../../README.md#contents) · [Session briefing](../using/briefing.md) →
