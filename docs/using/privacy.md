# Privacy

What agent-mem stores, what it never stores, and how to keep things out.

## The short version

- **Local.** One SQLite file in your user folder. No telemetry, no cloud, no server, no account.
- **No API key.** Facts are written by the agent's own CLI under your existing login.
- **A request stays with its vendor.** A Claude Code request is never sent to OpenAI, and the other way round.
- **Secrets are removed** before anything is stored.
- **Not encrypted.** Anyone who can read your home folder can read it.

## What is stored

| Stored | For |
| --- | --- |
| your requests and the agent's answers (cleaned, at most 16,000 characters each) | search, the briefing, facts |
| tool calls: which tool, the command or file, a cut version of the output, whether it failed | fix hints, file history, recipes |
| facts, decisions, rules, lessons written from them | reminders |
| the notes Claude Code and Codex keep themselves | shown to the other agents |

Tool output and inputs are trimmed after 30 days (`retention.payload_days`); the requests, answers and memories
stay until you delete them.

## What is never stored

- anything between `<private>` and `</private>`, in a prompt, a file, or tool output;
- secrets in known formats (API keys from Anthropic, OpenAI, GitHub, GitLab, Slack, Stripe, Google, AWS, npm, PyPI,
  Hugging Face; JWTs, bearer tokens, private keys, passwords in URLs, command lines, headers and assignments);
- the content of files matching `exclude_globs` (`.env`, `*.pem`, `*.key`, `id_rsa*`, `secrets/**`, … and your
  own); only that such a file was touched;
- web pages and third-party MCP output never go into the text the fact model reads;
- agent-mem's own processes and model calls;
- anything in projects that are excluded, opted out or paused.

The full list of patterns is on [Capture & Privacy](../architecture/capture-and-privacy.md#secret-formats).

## Keeping things out

| Want to keep out | Do |
| --- | --- |
| one piece of text | wrap it: `<private>the staging key is …</private>` |
| certain files | add globs to `exclude_globs` in `config.json` or `"exclude"` in the project's `.agent-mem.json` |
| a whole project | `{"capture": false}` in its `.agent-mem.json`, or its path in `excluded_projects` |
| everything for a while | `agent-mem pause --for 2h` (or until `agent-mem resume`) |
| one agent process | start it with `AGENT_MEM_DISABLE=1` |
| model calls | `"observe": {"enabled": false}`: requests are still recorded and searchable, no facts |
| the agents' own notes | `"native_notes": false` |

## What leaves your machine

| Traffic | When |
| --- | --- |
| model downloads (search model about 220 MB, spaCy model about 13 MB) | `setup --write`, `models install`, after a model change |
| your agent's CLI talking to its own provider | when facts are written, under your login, with the request's text |
| nothing else | mem0's telemetry is switched off and a test checks it |

> [!WARNING]
> Redaction catches known formats. A password in an unusual shape can slip through. For what must never be stored,
> use `<private>` or an exclude glob, and delete what slipped through with `agent-mem purge --id T12`.

---

← [Rules & lessons](rules-and-lessons.md) · [Contents](../../README.md#contents) · [Pause, export, purge](pause-export-purge.md) →
