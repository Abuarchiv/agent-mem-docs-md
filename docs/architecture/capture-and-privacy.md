# Capture & Privacy

What a hook stores from each event, what it never stores, how secrets are removed, and how a folder becomes a project.

Every event goes through one write path, `hooks/capture.py apply()`. Live hooks, spool replays and transcript
imports all use it. Memories from other sources (MCP, the observer) go through `memory/store.py add_memory()`,
which cleans text the same way. This page lists what is stored, what is not, and why.

## Before anything is written

The checks run in this order. The first that says no ends the event, and nothing at all is written. Not even the
project is registered.

| # | Check | Effect |
| --- | --- | --- |
| 1 | `AGENT_MEM_INTERNAL=1` (agent-mem's own processes and model calls) | hook returns at once |
| 2 | `AGENT_MEM_DISABLE=1` | hook returns at once |
| 3 | the tool is agent-mem's own (`agent-mem`, `agent_mem`, `agentmem`, `mem_search`, `mem_timeline`, `mem_get`, `mem_remember`, `mem_forget`, also as `mcp__…__mem_get`) | event skipped |
| 4 | `"capture": false` in `config.json` | nothing recorded |
| 5 | recording paused (`agent-mem pause [--for 2h]`) | nothing recorded until `resume` or the time is up |
| 6 | the project lies inside `~/.codex`, `~/.claude`, `~/.copilot` or `~/.config/opencode` (the agents' own background jobs) | nothing recorded; `~/.copilot/copilot-worktrees/<repo>` **is** recorded |
| 7 | the project is inside one of `excluded_projects` | nothing recorded |
| 8 | the project's `.agent-mem.json` says `{"capture": false}` | nothing recorded |

## What one event stores

The `events.payload` column holds a small JSON object. Every text in it has passed through
[`clean_text`](#cleaning-text):

| Field | When | Limit |
| --- | --- | --- |
| `raw_kind` | always: the agent's event name | 64 characters |
| `trust` | always: `user` for prompts and answers, `tool_local` for tools, `tool_external` for web and MCP tools | — |
| `source` | session start reason, compaction trigger, end reason | 64 |
| `prompt` | prompt events | `limits.prompt_chars` (16,000) |
| `answer` | stop events | `limits.answer_chars` (16,000) |
| `category` | tool events: `shell`, `edit`, `read`, `web`, `mcp`, `other` | — |
| `input` | the tool input as JSON | `limits.tool_input_chars` (4,000) |
| `output` | the tool output, denoised | `limits.tool_output_chars` (8,000) |
| `failed`, `exit_code` | when known | — |
| `error_sig`, `error_line` | failed tools: a stable signature and the first error line | 4,000 read |

Columns of `events` add `session_id`, `turn_id`, `ts`, `kind`, `tool`, `tool_use_id`, `command_key`, `files_json`,
`resolved` and the unique `dedupe_key`. Tool names and ids from the agent are cleaned and cut too (200, 128 and
64 characters).

Payloads are not kept forever. For events older than `retention.payload_days` (30) the indexer trims `payload` to
`raw_kind`, `category`, `error_line` and `trust` (`trimmed = 1`); prompt, answer, tool input and output go. The turn
row with its request, answer, files, commands and errors stays. Quarantine notes, which can hold a full payload,
are deleted after the same 30 days.

## Cleaning text

```python title="core/privacy.py"
def clean_text(text, limit):
    text = strip_control(strip_private(text))   # <private>…</private> → [private]; ANSI and control characters out
    text, _ = redact(text)                      # known secret formats → [REDACTED:<kind>]
    return truncate(text, limit)                # 60 % head, "…[truncated]…", the rest from the tail
```

### Private sections

Anything between `<private>` and `</private>` becomes `[private]` before it is stored. An unclosed `<private>`
hides the rest of the text. It works in prompts, in files the agent reads, in tool output, anywhere.

```text
Deploy with the staging key <private>sk-live-…</private> and run the smoke test
→ Deploy with the staging key [private] and run the smoke test
```

### Secret formats

Replaced with `[REDACTED:<kind>]`, most specific first:

| Kind | Matches |
| --- | --- |
| `private_key` | `-----BEGIN … PRIVATE KEY-----` up to its end line (or the end of the text) |
| `anthropic_key` | `sk-ant-…` |
| `openai_key` | `sk-…`, `sk-proj-…`, `sk-svcacct-…`, `sk-admin-…` |
| `github_token` | `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`, `github_pat_…` |
| `gitlab_token` | `glpat-…` |
| `slack_token` | `xoxa-`, `xoxb-`, `xoxp-`, `xoxo-`, `xoxs-`, `xoxr-…` |
| `stripe_key` | `sk_live_`, `sk_test_`, `rk_live_`, `rk_test_…` |
| `google_api_key` | `AIza` + 35 characters |
| `aws_access_key` | `AKIA…`, `ASIA…` (20 characters) |
| `npm_token`, `pypi_token`, `hf_token` | `npm_…`, `pypi-…`, `hf_…` |
| `jwt` | three base64url parts starting `eyJ` |
| `bearer` | `Bearer <token>` |
| `url_credentials` | `scheme://user:password@` → `scheme://[REDACTED:url_credentials]@` |

Replaced with `[REDACTED:secret]`, keeping the part before the value:

| Kind | Matches |
| --- | --- |
| command-line passwords | `mysql … -p<pw>`, `--password <pw>`, `--pass <pw>`, `sshpass -p <pw>`, `docker/podman/helm login … -p <pw>`, `redis-cli … -a <pw>`, `-u user:<pw>` |
| headers | whole `Cookie:` / `Set-Cookie:` values; `Authorization: Basic/Digest/Token <value>` |
| AWS secret | 40 base64 characters after `aws … secret` |
| a password in words | `password is …`, `Passwort ist …`, `the password for the db is …`, `pin: …`, if the value looks like a password (6+ characters with a digit, a symbol or mixed case) |
| assignments | `*api_key*`, `*secret*`, `*token*`, `*password*`, `*passwd*`, `*pwd*`, `*passphrase*`, `*client_secret*`, `*private_key*`, `*access_key*`, `*account_key*`, `*auth_token*`, `*credential*` followed by `=`, `:` or a JSON `"…": "…"` |

Assignments keep values that are clearly not secrets: digits only, `true`, `false`, `null`, `none`,
`undefined`, `required`, `optional`, `string`, and references starting with `$`, `{{`, `<` or `[REDACTED`.
`das Passwort ist nicht gesetzt` stays; `pw = getpass()` stays, because it names where the password comes from.

> [!WARNING]
> Redaction catches known formats. A password in an unusual place can slip through. Use `<private>` for what must
> never be stored, exclude secret files by glob, and remember that the database is **not encrypted**.

### Long tool output

Before cleaning, tool output longer than its limit is denoised: the first 15 lines, up to 40 lines from the middle
that look like errors (`error`, `exception`, `traceback`, `failed`, `fatal`, `panic`, `denied`, `not found`,
`cannot`, `undefined`, `ERR!`, `assert`, `✗`, `✘`), and the last 15 lines. Then it is truncated like any text.

## Excluded files

`exclude_globs` (plus the project's `.agent-mem.json` `"exclude"`) name files whose **content** is never stored.
The defaults:

```json
[".env", ".env.*", "*.pem", "*.key", "*.p12", "*.pfx", "id_rsa*", "id_ed25519*", "id_ecdsa*",
 ".npmrc", ".netrc", ".pypirc", "secrets/**", "**/secrets/**", "*.kdbx", "credentials*"]
```

A glob matches the file name or the whole path; `**/x` also matches `x` at the top. When a tool touches an excluded
file:

- its output becomes `[excluded by privacy rules]`;
- an edit's input becomes `{"files": [...], "content": "[excluded by privacy rules]"}`;
- the path itself is still recorded, so "`.env` was edited" is known; what it holds is not.

## How a folder becomes a project

`core/identity.py resolve(cwd)` decides which project an event belongs to. Git is asked only the first time a
folder is seen; after that `project_paths` maps the folder directly.

| Case | Project id from |
| --- | --- |
| a git repository with `origin` | the normalized remote: `git@github.com:Me/App.git` and `https://github.com/me/app` are both `github.com/me/app` |
| a git repository without a remote, with commits | its root commit (stays the same across moves, renames, clones and worktrees) |
| a git repository without commits, or a plain folder | the path of the repository's git directory, or of the folder |
| a worktree | the same as its repository (shared git directory) |

The id is the first 16 hex characters of the SHA-256 of that source. The name shown is the last part of the remote,
or the folder name.

- **Nested repositories stay separate.** A parent never swallows a repository below it. `~/work` and
  `~/work/app` are two projects. A turn asked in `~/work` whose tools work in `~/work/app` moves to `app`, and so
  does its session.
- **Templates.** Two copies of a template share their root commit. If that id is already held by another existing
  repository, the path decides instead.
- **Aliases.** A project first seen before its first commit or before it got a remote is the same project
  afterwards. The indexer records the alias (two git calls per repository, outside the hook path).
- **Branch.** The current branch is read from `.git/HEAD` and stored with the session.

Searches cover a project, its nested projects and the projects around it, never the home folder. See
[Search Architecture](search.md#scope).

## What the signals read from a tool call

`core/signals.py analyze_tool()` turns a tool call into a few facts without a model:

| Signal | How |
| --- | --- |
| category | by tool name: `shell` (`Bash`, `exec_command`, `run_terminal_cmd`, `powershell`, `shell`, …), `edit` (`Edit`, `Write`, `MultiEdit`, `apply_patch`, `str_replace_editor`, `NotebookEdit`, …), `read` (`Read`, `view`, `read_file`, `cat`, …), `web` (`WebFetch`, `WebSearch`, `fetch`, `browser`, …), `mcp` (`mcp__…`), else `other` |
| command | the shell command from the input |
| command kind | `test` (pytest, jest, vitest, go test, cargo test, playwright, …), `build` (build, tsc, make, cargo build, webpack, vite build, …), `lint` (ruff, eslint, mypy, pyright, clippy, biome, typecheck, …), `install` (`npm i`, `pnpm add`, `pip install`, `uv add`, `uv sync`, `cargo add`, `go get`, …), `commit`, `revert` |
| command key | the working part: `cd x && UV_CACHE_DIR=/tmp uv run pytest -q 2>&1 \| tail` → `uv run pytest`; for multi-command tools (`npm`, `git`, `uv`, `docker`, `cargo`, `make`, …) the subcommand is kept |
| files | from the input (`file_path`, `path`, …), from patches (`*** Update File:`), from diffs (`+++ b/`) |
| packages | names added by an install command |
| reverted files | from `git checkout -- <files>`, `git restore <files>`; `git revert` and `git reset --hard` revert everything edited |
| error signature | the first error line with paths, `file:line`, hex ids, numbers, quoted names and extra spaces removed, so the same error matches across runs |

These signals feed recipes, rules, dead ends, the commands the briefing lists, entities and the association graph.

## Storage

- One SQLite file, `memory.db`, in the data folder (`agent-mem paths`). On macOS and Linux the folder is created
  with owner-only permissions (`0700`), `mem0/` too. `agent-mem doctor` checks this ("data folder private").
- **Not encrypted.** Anyone who can read your home folder can read it.
- The only network traffic is model downloads (`agent-mem models install`, first `setup`) and your agent's own CLI
  talking to its own provider under your login. mem0's telemetry is switched off (`MEM0_TELEMETRY=False`, patched in
  its modules), and a test checks it.
- `agent-mem purge` deletes from the database, the search index, the vector index and the spool. See
  [Pause, Export, Purge](../using/pause-export-purge.md).

---

← [Learning & Ranking](learning-and-ranking.md) · [Contents](../../README.md#contents) · [Token Accounting](token-accounting.md) →
