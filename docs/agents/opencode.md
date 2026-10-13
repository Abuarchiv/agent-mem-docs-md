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

OpenCode has no command hooks; it loads TypeScript plugins. agent-mem's plugin (typed against
`@opencode-ai/plugin` 1.18) maps OpenCode's hooks to `agent-mem hook opencode <event>`:

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

Each call spawns the command with the JSON on stdin and kills it after 2 seconds. Any error resolves to "no
context"; the only error the plugin throws on purpose is an enabled rule. Calls to agent-mem's own tools are
skipped.

## What OpenCode does not get

- **No panel.** OpenCode has no channel for a message to you.
- **No facts by default.** No OpenCode CLI is used for extraction. OpenCode requests are saved word for word,
  unless you add `"ollama"` to `observe.providers` (or set `observe.cross_vendor: true` to let another installed
  CLI take them).
- **No token accounting.** OpenCode's work per request is not measured.

> [!WARNING]
> Run OpenCode sessions one at a time per repository. Two at once can collide on OpenCode's own snapshot lock; this is
> an OpenCode limitation, not agent-mem's.

Tested with OpenCode 1.18.

---

← [Copilot CLI & app](copilot.md) · [Contents](../../README.md#contents) · [Session briefing](../using/briefing.md) →
