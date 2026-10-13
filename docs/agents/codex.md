# Codex

How agent-mem connects to the Codex CLI, the trust step, and what Codex shows.

## Connect

1. Write hooks and the MCP server:

   ```sh
   agent-mem setup codex --write
   ```

   Hooks go to `~/.codex/hooks.json`, the MCP server to `~/.codex/config.toml`:

   ```toml
   [mcp_servers.agent-mem]
   command = "/Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem"
   args = ["mcp"]
   ```

2. **Trust the hooks once.** Start `codex`, run `/hooks` and trust the agent-mem hooks. Codex requires this review
   for new and changed hooks, and asks again after an upgrade that changed them.

`CODEX_HOME` moves `~/.codex`. A Codex plugin also exists (`codex plugin marketplace add Abuarchiv/agent-mem`,
`codex plugin add agent-mem@agent-mem`); it needs the same trust step.

## Hooks

| Event | Timeout | agent-mem does |
| --- | --- | --- |
| `SessionStart` | 10 s, `additionalContextLimit: 8000` | briefing, panel |
| `SubagentStart` | 10 s | briefing for the subagent |
| `UserPromptSubmit` | 10 s, `additionalContextLimit: 8000` | new turn, rules, confident matches |
| `PreToolUse` | 10 s | rule note or deny, file history (also for `sed -n …` reads of a file) |
| `PostToolUse` | 10 s, `additionalContextLimit: 8000` | record, recipes, fix hints |
| `Stop` | 10 s | close the turn, start the observer |
| `SubagentStop` | 10 s | recorded |
| `PreCompact` (`PostCompact`, `Compact`) | 10 s | recorded |
| `SessionEnd` | **3 s** | close the session |

Every entry carries `"statusMessage": "Agent Mem"`. Codex has no matchers here: every tool goes through the hook.
There is no `PostToolUseFailure`; failures arrive as `PostToolUse`.

## What Codex does differently

- **No exit codes.** Codex 0.162 sends a shell call's output without its exit code. agent-mem reads failure from
  clear signs in the output: a traceback, `N failed`, a line starting with `error:` or `fatal:`, `npm ERR!`,
  `command not found`.
- **Context size.** Without `additionalContextLimit` Codex passes about 2,500 tokens per hook. The setting is 8,000
  on the three large events; agent-mem caps those at 7,800 tokens and the rest at 2,400.
- **Panel.** Shown as a plain warning, without colours. Nothing is shown for `SessionEnd`.

## Facts

```bash
codex exec --ephemeral --ignore-user-config --disable hooks --disable memories \
           -s read-only --skip-git-repo-check -m gpt-6-luna -c model_reasoning_effort=low
```

Under your Codex login, read-only, without your config, hooks or memories. `observe.codex_model` changes the model.

## Codex's own memory

Codex writes one summary per session to `~/.codex/memories/rollout_summaries/`. agent-mem reads them as notes and
shows them to Claude Code, Copilot and OpenCode, never back to Codex. Codex prunes old summaries by age; agent-mem
keeps them.

## Token accounting

From `~/.codex/sessions/YYYY/MM/DD/*<session>.jsonl`: each `token_count` event's fresh input and output.

> [!WARNING]
> Codex records nothing until you trust the hooks in `/hooks` (unless your terminal setup already trusts them).
> `agent-mem doctor` shows no Codex recording until the first event arrives.

---

← [Claude Code](claude-code.md) · [Contents](../../README.md#contents) · [Copilot CLI & app](copilot.md) →
