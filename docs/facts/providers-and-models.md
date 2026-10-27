# Providers & models

Which CLI and model write facts for which agent, how to change it, and what it costs.

agent-mem has no API key and no model of its own. It starts the CLI you already use, under your login, with no
tools, no MCP servers, no settings and no saved session, in an empty temporary folder.

![Claude Code requests to claude -p with haiku, Codex requests to codex exec with gpt-6-luna, Copilot requests to copilot -s auto, OpenCode requests to opencode run with the free big-pickle model; Ollama only when added.](../../assets/diagrams/provider-chain.svg)

<sub>[Excalidraw scene](../../assets/diagrams/provider-chain.excalidraw)</sub>

| Request came from | Model | Command |
| --- | --- | --- |
| Claude Code | Claude Haiku | `claude -p --restricted --setting-sources "" --tools "" --strict-mcp-config --no-session-persistence --model haiku` |
| Codex | GPT-6-Luna, low effort | `codex exec --ephemeral --ignore-user-config --disable hooks --disable memories -s read-only -m gpt-6-luna -c model_reasoning_effort=low` |
| Copilot CLI / app | auto, efficiency tier (resolves to GPT-6-Luna) | `copilot -s --no-custom-instructions --disable-builtin-mcps --model auto --auto-tier efficiency` |
| OpenCode | Big Pickle, free | `opencode run --standalone --agent plan --model opencode/big-pickle` |

**A request never leaves its vendor.** A Claude Code request is not sent to OpenAI, a Codex request not to
Anthropic. Each uses the cheapest model of its CLI. With only one CLI installed, its requests get facts and the
others are saved word for word.

## Changing models

```json title="config.json"
{
  "observe": {
    "claude_model": "sonnet",
    "codex_model": "gpt-6",
    "copilot_model": "auto"
  }
}
```

If your plan does not offer a model, or the CLI is too old for the flag, the call is tried once more with the CLI's
own default. `agent-mem doctor` and the session panel name the model in use: `Facts written by haiku (claude)`.

## Ollama, locally

```json title="config.json"
{ "observe": { "providers": ["opencode", "claude", "codex", "copilot", "ollama"], "ollama_model": "qwen3.5:4b-mlx" } }
```

Ollama is off by default because local models load the machine. Added, it is the fallback for any request whose
own CLI is missing, paused or fails. It is not counted against the daily limit.

## Letting any CLI take any request

```json title="config.json"
{ "observe": { "cross_vendor": true } }
```

Then the first installed CLI in `observe.providers` takes every request, whatever agent it came from. Off by
default, because it sends one vendor's conversation to another.

## Cost

Facts use the subscription or plan you already have, through the CLI's own login.

| Per batch of up to 5 requests | Calls |
| --- | --- |
| facts | 1 |
| titles, session summary, lessons | 1 |
| per project and day | 1 review, up to 3 graph batches of 40 memories |

`observe.daily_limit` (300) caps model calls per day. `agent-mem observe --status` shows today's calls per
provider. A usage limit pauses that provider for `observe.pause_minutes` (60); requests wait rather than being lost.

> [!NOTE]
> agent-mem never reads, copies or stores a token. Copilot gets a private `COPILOT_HOME` per call (its login file is
> copied in when one exists), so the call is not saved as a session and none of your hooks or plugins run.

---

← [How facts are written](how-facts-are-written.md) · [Contents](../../README.md#contents) · [config.json](../configuration/config-json.md) →
