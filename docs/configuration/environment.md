# Environment variables

The variables agent-mem reads, and the agents' own variables it follows.

A flag counts as set when its value is `1`, `true`, `yes` or `on` (any case).

## agent-mem's own

| Variable | Read by | Effect |
| --- | --- | --- |
| `AGENT_MEM_DATA_DIR` | everything | the data folder instead of the system default; every process agent-mem starts gets it |
| `AGENT_MEM_DISABLE` | hooks | the hook returns at once: nothing recorded, nothing injected |
| `AGENT_MEM_INTERNAL` | hooks | set by agent-mem for its own processes and model calls, so they are never recorded; do not set it yourself |
| `AGENT_MEM_NO_SPAWN` | hooks, setup | hooks start no observer, indexer or viewer; setup writes the login item without starting it |
| `AGENT_MEM_PROJECT_DIR` | MCP server | the project to search in, when the agent does not set `CLAUDE_PROJECT_DIR` |
| `AGENT_MEM_DEBUG` | CLI | print the full traceback instead of a one-line explanation |
| `AGENT_MEM_FAKE_LLM` | observer, tests | path of a JSON list of canned model answers for the `fake` provider |
| `AGENT_MEM_EXE` | setup, upgrade | the agent-mem binary written into hook files. The Docker launcher sets it to itself, since setup runs inside the container |
| `AGENT_MEM_MODEL_DIR` | search, observer | folder of the embedding and spaCy models instead of `<data>/models`; the Docker image keeps them outside the data volume, so a new image brings new ones |
| `AGENT_MEM_IN_DOCKER` | viewer, panel, doctor | flag set by the Docker image: login hints name `agent-mem login <agent>` (the fact-writing CLIs live in the container), and `AGENT_MEM_VIEWER_BIND` applies |
| `AGENT_MEM_VIEWER_BIND` | viewer | address the viewer listens on, only together with `AGENT_MEM_IN_DOCKER` (the image sets `0.0.0.0`; Docker publishes the port to the host's 127.0.0.1 only). Elsewhere the viewer always stays on 127.0.0.1 |

The data folder defaults:

| OS | Default |
| --- | --- |
| macOS | `~/Library/Application Support/agent-mem` |
| Linux | `$XDG_DATA_HOME/agent-mem`, else `~/.local/share/agent-mem` |
| Windows | `%LOCALAPPDATA%\agent-mem`, else `~/AppData/Local/agent-mem` |

`agent-mem --data-dir <folder> <command>` overrides it for one command.

## The agents' variables it follows

| Variable | Used for |
| --- | --- |
| `CLAUDE_CONFIG_DIR` | where Claude Code's `settings.json`, transcripts and `memory/*.md` notes live (default `~/.claude`) |
| `CLAUDE_PROJECT_DIR` | the MCP server's project, set by Claude Code |
| `CODEX_HOME` | Codex's `hooks.json`, `config.toml`, sessions and memories (default `~/.codex`) |
| `COPILOT_HOME` | Copilot's hooks, `mcp-config.json`, session state and login (default `~/.copilot`) |
| `XDG_CONFIG_HOME` | OpenCode's plugin and `opencode.json`, the systemd user unit (default `~/.config`) |
| `XDG_DATA_HOME` | the data folder on Linux |
| `LOCALAPPDATA`, `APPDATA` | the data folder and the Startup folder on Windows |
| `NO_COLOR` | plain text in the Claude Code panel |
| `PATH` | finding `claude`, `codex`, `copilot`, `ollama`, `uv`; copied into the viewer's login item |

## What agent-mem sets for others

| Process | Environment |
| --- | --- |
| observer, indexer, viewer started by a hook | `AGENT_MEM_INTERNAL=1`, `AGENT_MEM_DATA_DIR` |
| `claude -p`, `codex exec`, `copilot -s` for facts | `AGENT_MEM_INTERNAL=1`; every `CLAUDECODE*` variable removed so Claude Code does not think it runs nested; Copilot gets a private temporary `COPILOT_HOME` |
| mem0 | `MEM0_TELEMETRY=False`, `MEM0_DIR=<data>/mem0` |
| viewer login item | `AGENT_MEM_DATA_DIR`, `AGENT_MEM_INTERNAL=1`, the `PATH` from setup |

---

← [CLI](cli.md) · [Contents](../../README.md#contents) · [Architecture Overview](../architecture/overview.md) →
