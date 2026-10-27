# Installation

Install agent-mem, connect your agents, and check the result.

## Requirements

- Python 3.11 to 3.14 (uv installs one if needed)
- [uv](https://docs.astral.sh/uv/) or pipx
- At least one agent: Claude Code, Codex CLI, GitHub Copilot CLI (or the Copilot app on macOS), OpenCode
- For background facts: the CLI of the agent, logged in (`claude`, `codex`, `copilot`, `opencode`). Nothing else: no API key.

## Install
**uv (recommended)**

```sh
uv tool install "agent-mem[semantic,observer] @ git+https://github.com/Abuarchiv/agent-mem"
agent-mem setup all --write
agent-mem doctor
```

**Install script (macOS, Linux)**

```sh
curl -LsSf https://raw.githubusercontent.com/Abuarchiv/agent-mem/main/install.sh | sh
```

Installs uv first if it is missing, then agent-mem, then runs `agent-mem setup all --write` on a fresh machine.
Running it again upgrades.

**Install script (Windows)**

```powershell
irm https://raw.githubusercontent.com/Abuarchiv/agent-mem/main/install.ps1 | iex
```

Installs uv and agent-mem like the macOS script. Search by meaning needs the Microsoft Visual C++ runtime; a fresh
Windows often lacks it, so the script installs it when `msvcp140.dll` or `vcruntime140_1.dll` is missing (Windows
asks for administrator rights once). Skipped, search by words still works and `agent-mem doctor` names what to
install. Git for Windows is required (`winget install Git.Git`).

**Docker (macOS, Linux)**

```sh
curl -LsSf https://raw.githubusercontent.com/Abuarchiv/agent-mem/main/install-docker.sh | sh
agent-mem login claude    # once per agent that should write facts: claude, codex, copilot, opencode
```

For a computer with nothing but Docker. Your agents stay where they are; agent-mem runs in a container named
`agent-mem`. The script builds the image with the agent CLIs it finds (1.5 to 3 GB, a few minutes the first time),
starts the container (it restarts with Docker), writes a small `agent-mem` command to `~/.local/bin` that runs
everything in the container, and connects every agent it finds.

- Memory lives in the Docker volume `agent-mem-data`. The container reads your home folder (transcripts, git) but
  cannot change it.
- Facts are written by the CLIs inside the container, so log each in once with `agent-mem login <agent>`. Until
  then requests are recorded and searchable and wait for facts.
- `agent-mem view` opens http://localhost:37778/; another port: `AGENT_MEM_PORT=37790` before the install line.
- With Docker closed, hooks end silently and your agent works without memory. Each hook is a `docker exec`: about
  150 ms per event instead of about 70 ms.

**pipx**

```sh
pipx install "agent-mem[semantic,observer] @ git+https://github.com/Abuarchiv/agent-mem"
agent-mem setup all --write
```

`agent-mem upgrade` only works for `uv tool` installs; with pipx, upgrade with pipx and run `agent-mem doctor --fix`.

**Claude Code plugin**

```text
/plugin marketplace add Abuarchiv/agent-mem
/plugin install agent-mem@agent-mem
```

The plugin only registers the hooks and the MCP server. Install the `agent-mem` command first with the `uv tool
install` line, and skip `setup --write` (or every hook runs twice). Without the command, every hook fails with
"command not found".

### Agent in a container (Docker, Dev Containers)

If your agent itself runs in a container, install agent-mem into that image and keep its data on a volume:

```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/
USER node
ENV PATH=/home/node/.local/bin:$PATH AGENT_MEM_DATA_DIR=/home/node/.agent-mem
RUN uv tool install --python 3.13 "agent-mem[semantic,observer] @ git+https://github.com/Abuarchiv/agent-mem" \
 && agent-mem setup claude --write
```

```sh
docker run -it -v agent-mem-data:/home/node/.agent-mem -v claude-home:/home/node/.claude <your-image>
```

Log in to the agent inside the container once; agent-mem writes facts through that login. The viewer listens on
127.0.0.1 inside the container, so open it there.

> [!WARNING]
> agent-mem is not on PyPI. Never install it by its bare name (`pip install agent-mem`): a package registered there
> under that name would not be this one.

## Three sizes

| Install | Size | You get |
| --- | --- | --- |
| `agent-mem` | about 10 MB | recording, search by words, reminders for the agent, MCP server, requests saved word for word |
| `agent-mem[semantic]` | plus 220 MB model | search by meaning (`paraphrase-multilingual-MiniLM-L12-v2`, ONNX, no PyTorch) |
| `agent-mem[semantic,observer]` | plus about 400 MB | background facts, work log, lessons, daily review (mem0, Qdrant local mode, spaCy) |

Add an extra later with `agent-mem upgrade --extras semantic observer`.

## What setup writes

`agent-mem setup all --write` connects every agent it finds on `PATH`. Name one instead of `all` to connect just
that one. Without `--write` it only prints the steps.
1. **Hooks** with the absolute path of the `agent-mem` command, so sessions started from an editor or a launcher
   with a short `PATH` find it:
   - Claude Code: `~/.claude/settings.json`
   - Codex: `~/.codex/hooks.json`
   - Copilot: `~/.copilot/hooks/agent-mem.json`
   - OpenCode: `~/.config/opencode/plugins/agent-mem.ts`
2. **The MCP server**: `claude mcp add --scope user agent-mem`, `[mcp_servers.agent-mem]` in
   `~/.codex/config.toml`, `~/.copilot/mcp-config.json`, `opencode.json`.
3. **Models**: the search model (about 220 MB) and the spaCy topic model (about 13 MB) into the data folder. Skip
   with `--no-models`.
4. **The viewer** as a login item, started now: launchd on macOS, a systemd user unit on Linux, the Startup folder
   on Windows.
Every file is written atomically, the previous version is kept as `<file>.agent-mem-backup`, and your own hooks
stay. A config file that is not valid JSON stops setup with "Nothing was changed".

```text title="agent-mem setup all --write (excerpt)"
claude: hooks and MCP server registered with /Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem.
  wrote /Users/you/.claude/settings.json
  facts: written in the background by `claude` (haiku), with your existing login.
codex: hooks and MCP server registered with /Users/you/.local/share/uv/tools/agent-mem/bin/agent-mem.
  wrote /Users/you/.codex/hooks.json
  wrote /Users/you/.codex/config.toml
  Codex asks you to trust the hooks once: start codex and run /hooks.
  facts: written in the background by `codex` (gpt-6-luna), with your existing login.
search model sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2 ready.
viewer: http://localhost:37778/, starts at login (/Users/you/Library/LaunchAgents/io.github.abuarchiv.agent-mem.viewer.plist).

Start a session, then run `agent-mem doctor`. `agent-mem view` opens the viewer.
```

## One thing only you can do

- **Codex:** start `codex`, run `/hooks` and trust the agent-mem hooks once. Codex asks again after an upgrade
  changed them.
- **Facts:** each agent's CLI must be logged in once (`claude`, `codex`, `copilot`). agent-mem uses that login
  and never reads or stores a token.

## Check

```sh
agent-mem doctor
```

Every line should be `✓` or `!`. A `!` names the next step; see [doctor](../configuration/cli.md#doctor). After your
first session, `agent-mem status` shows what was saved.

## Where things are

```sh
agent-mem paths
```

| | macOS | Linux | Windows |
| --- | --- | --- | --- |
| Data | `~/Library/Application Support/agent-mem` | `~/.local/share/agent-mem` | `%LOCALAPPDATA%\agent-mem` |
| Command (uv) | `~/.local/bin/agent-mem` | `~/.local/bin/agent-mem` | `%USERPROFILE%\.local\bin\agent-mem.exe` |

`AGENT_MEM_DATA_DIR` moves the data folder.

---

← [agent-mem](../../README.md) · [Contents](../../README.md#contents) · [Your first session](first-session.md) →
