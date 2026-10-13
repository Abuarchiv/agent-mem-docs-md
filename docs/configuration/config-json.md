# config.json

Every setting agent-mem reads, its default, its allowed range and what it changes.

agent-mem reads one file, `config.json` in the data folder (`agent-mem paths` shows it as "Settings"). It is
optional. Every key you leave out keeps its default, and you only write the keys you want to change:

```json title="config.json"
{
  "budgets": { "session_start": 4000 },
  "observe": { "providers": ["claude", "codex", "copilot", "ollama"] },
  "viewer": { "port": 37800 }
}
```

The file is read by every process at start; there is nothing to restart. A running viewer keeps its port until it
is restarted, and a running MCP server keeps its settings until the agent restarts it.

## Invalid values never break anything

| Problem | What happens | Where you see it |
| --- | --- | --- |
| the file is not valid JSON, or not an object | all defaults | `config: … is not valid JSON` |
| an unknown key | ignored | `config: unknown key 'observe.provider' ignored` |
| a group that is not an object | defaults for that group | `config: 'budgets' must be an object` |
| a value of the wrong type (a string for a number, a negative number) | default for that key | `config: invalid value for 'viewer.port', default kept` |
| a number outside its range | default for that key | `config: 'hook_deadline_ms' must be between 100 and 10000, default kept` |

The warnings appear in `agent-mem doctor` and, at most two of them, in the panel at session start. The ranges
exist because a well-typed value can still break agent-mem silently: a deadline of 0 would spool every event,
`stdin_bytes: 0` would record nothing, `duplicate_min_cosine: 0` would retire every older fact.

## Top level

| Key | Default | Range / type | Effect |
| --- | --- | --- | --- |
| `capture` | `true` | bool | `false`: nothing is recorded anywhere; recall still works |
| `native_notes` | `true` | bool | read Claude Code's `memory/*.md` and Codex's rollout summaries as notes ([details](../architecture/background-processes.md#native-notes)) |
| `hook_deadline_ms` | `400` | 100–10,000 | how long a hook works before it spools the event and prints nothing |
| `exclude_globs` | 16 patterns (below) | list of strings | files whose content is never stored; replaces the default list |
| `excluded_projects` | `[]` | list of paths | nothing in these folders is recorded |

```json title="exclude_globs default"
[".env", ".env.*", "*.pem", "*.key", "*.p12", "*.pfx", "id_rsa*", "id_ed25519*", "id_ecdsa*",
 ".npmrc", ".netrc", ".pypirc", "secrets/**", "**/secrets/**", "*.kdbx", "credentials*"]
```

> [!WARNING]
> `exclude_globs` **replaces** the list. To add a pattern, copy the defaults and append yours, or put project-specific
> patterns in that project's [`.agent-mem.json`](project-settings.md), which adds to the list.

## budgets

Estimated tokens per injected block (a quarter token per ASCII character, one per other character). Each is also
capped by what the agent passes in one piece ([per-agent caps](../architecture/context-injection.md#per-agent-caps)).

| Key | Default | Range | Block |
| --- | --- | --- | --- |
| `budgets.session_start` | `8000` | 0–200,000 | the briefing at session start and for subagents |
| `budgets.prompt` | `2400` | 0–100,000 | matching memories for a prompt |
| `budgets.failure` | `400` | 0–20,000 | the fix hint after a failed command |
| `budgets.warning` | `300` | 0–20,000 | a rule note before a tool |
| `budgets.file_history` | `1200` | 0–20,000 | file history before a read or edit |
| `budgets.compact` | `4000` | 0–200,000 | the block after compaction |

`0` turns a block off. `budgets.session_start: 2000` gives about a quarter of the default briefing.

## retrieval

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `retrieval.inject_min_score` | `0.35` | 0–1 | score a hit needs to be injected or listed as confident |
| `retrieval.min_term_coverage` | `0.5` | 0–1 | share of the question's words a word-match hit must contain |
| `retrieval.max_per_session` | `3` | 1–1,000 | turns per session in one result list |
| `retrieval.rrf_k` | `60` | 1–1,000 | rank fusion constant; larger flattens differences between ranks |
| `retrieval.current_project_boost` | `1.2` | ≥ 0 | multiplier for hits from the current project |
| `retrieval.min_vector_score` | `0.3` | 0–1 | cosine floor for meaning-based candidates (calibrated for MiniLM) |

Lower `inject_min_score` and `min_term_coverage` for more context and more noise; raise them for less. See
[Search Architecture](../architecture/search.md).

## retention

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `retention.payload_days` | `30` | 1–36,500 | after this, event payloads are trimmed to kind, category, error line and trust; quarantine notes are deleted |
| `retention.access_days` | `180` | 1–36,500 | after this, `shown`/`warned`/`injected` access rows and model-call logs are deleted |
| `retention.max_db_mb` | `1024` | 64–1,048,576 | above this size the oldest payloads are trimmed and the database vacuumed, once a day at most |
| `retention.stale_turn_minutes` | `30` | 1–10,080 | an open turn without activity is closed as `unknown` after this |

## limits

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `limits.prompt_chars` | `16000` | 200–1,000,000 | stored characters of a prompt |
| `limits.answer_chars` | `16000` | 200–1,000,000 | stored characters of an answer |
| `limits.tool_output_chars` | `8000` | 200–1,000,000 | stored characters of a tool's output (head, error lines, tail) |
| `limits.tool_input_chars` | `4000` | 200–1,000,000 | stored characters of a tool's input |
| `limits.stdin_bytes` | `4194304` (4 MiB) | 64 KiB–64 MiB | a larger hook payload is refused and noted in quarantine |

## rules

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `rules.auto_enable` | `false` | bool | `true`: a rule switches on by itself after `min_corrections` and then **blocks** the command |
| `rules.min_corrections` | `2` | 1–100 | how often you must state "use X instead of Y" before a preference is stored |
| `rules.expire_days` | `90` | 1–36,500 | an enabled rule that has not fired for this long is switched off |

## semantic

Search by meaning; needs the `[semantic]` extra.

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `semantic.enabled` | `true` | bool | `false`: word search and graph only |
| `semantic.model` | `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` | fastembed model name | the embedder; changing it re-embeds everything and rebuilds the mem0 index |
| `semantic.batch_size` | `32` | 1–1,024 | texts per embedding batch |
| `semantic.duplicate_min_cosine` | `0.90` | 0.8–1 | an older observer or agent memory this similar to a newer one is retired; `1` turns it off |

E5 models (`intfloat/multilingual-e5-*`) get their `query:` and `passage:` prefixes automatically.

## observe

The background fact writer; needs the `[observer]` extra. See [Observer Pipeline](../architecture/observer-pipeline.md).

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `observe.enabled` | `true` | bool | `false`: no model calls at all; requests are still recorded and searchable, turns are not queued |
| `observe.providers` | `["claude", "codex", "copilot"]` | `claude`, `codex`, `copilot`, `ollama` | stages that may write facts, in order |
| `observe.cross_vendor` | `false` | bool | `true`: any installed CLI may take any agent's turns; `false`: a turn only goes to its own vendor (and Ollama) |
| `observe.claude_model` | `"haiku"` | string | `claude -p --model …` |
| `observe.codex_model` | `"gpt-6-luna"` | string | `codex exec -m …` (low reasoning effort) |
| `observe.copilot_model` | `"auto"` | string | `copilot -s --model …`; `auto` adds `--auto-tier efficiency` |
| `observe.ollama_model` | `"qwen3.5:4b-mlx"` | string | Ollama model, when `ollama` is in `providers` |
| `observe.entity_model` | `"en_core_web_sm"` | spaCy model name | topic links for facts; use the language of your facts (`de_core_news_sm`), then `agent-mem models install` |
| `observe.daily_limit` | `300` | ≥ 0 | model calls per UTC day; over it, turns wait until midnight UTC (Ollama is not limited) |
| `observe.idle_seconds` | `90` | ≥ 0 | the observer exits after this long without work |
| `observe.lease_seconds` | `900` | 60–86,400 | how long a worker holds a turn before another may take it |
| `observe.bundle_turns` | `5` | 1–50 | turns of one session per extraction call |
| `observe.bundle_tokens` | `20000` | 500–200,000 | size of one extraction call; larger bundles are split at turn boundaries |
| `observe.tool_output_chars` | `1200` | 100–100,000 | characters of each tool output the model sees |
| `observe.turn_tool_chars` | `12000` | 0–1,000,000 | tool output characters per turn in total; `0` sends none |
| `observe.pause_minutes` | `60` | ≥ 0 | a stage that hit a usage limit or lost its login pauses this long |

A model name the CLI refuses (your plan does not offer it, the CLI is too old for the flag) is retried once with
the CLI's own default.

## viewer

| Key | Default | Range | Effect |
| --- | --- | --- | --- |
| `viewer.enabled` | `true` | bool | `false`: the hooks never start the viewer and setup writes no login item |
| `viewer.port` | `37778` | 1,024–65,535 | the port on 127.0.0.1 |
| `viewer.refresh_seconds` | `10` | 2–3,600 | how often an open tab reloads |

## A complete file with every default

```json title="config.json (all defaults)"
{
  "capture": true,
  "native_notes": true,
  "hook_deadline_ms": 400,
  "exclude_globs": [".env", ".env.*", "*.pem", "*.key", "*.p12", "*.pfx", "id_rsa*", "id_ed25519*", "id_ecdsa*",
                    ".npmrc", ".netrc", ".pypirc", "secrets/**", "**/secrets/**", "*.kdbx", "credentials*"],
  "excluded_projects": [],
  "budgets": {"session_start": 8000, "prompt": 2400, "failure": 400, "warning": 300, "file_history": 1200, "compact": 4000},
  "retrieval": {"inject_min_score": 0.35, "min_term_coverage": 0.5, "max_per_session": 3, "rrf_k": 60,
                "current_project_boost": 1.2, "min_vector_score": 0.3},
  "retention": {"payload_days": 30, "access_days": 180, "max_db_mb": 1024, "stale_turn_minutes": 30},
  "limits": {"prompt_chars": 16000, "answer_chars": 16000, "tool_output_chars": 8000, "tool_input_chars": 4000,
             "stdin_bytes": 4194304},
  "rules": {"auto_enable": false, "min_corrections": 2, "expire_days": 90},
  "semantic": {"enabled": true, "model": "sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2",
               "batch_size": 32, "duplicate_min_cosine": 0.9},
  "observe": {"enabled": true, "providers": ["claude", "codex", "copilot"], "cross_vendor": false,
              "claude_model": "haiku", "codex_model": "gpt-6-luna", "copilot_model": "auto",
              "ollama_model": "qwen3.5:4b-mlx", "entity_model": "en_core_web_sm", "daily_limit": 300,
              "idle_seconds": 90, "lease_seconds": 900, "bundle_turns": 5, "bundle_tokens": 20000,
              "tool_output_chars": 1200, "turn_tool_chars": 12000, "pause_minutes": 60},
  "viewer": {"enabled": true, "port": 37778, "refresh_seconds": 10}
}
```

`data_dir` and `warnings` are not settable in the file. The data folder comes from `--data-dir`,
`AGENT_MEM_DATA_DIR` or the system default ([Environment](environment.md)).

---

← [Providers & models](../facts/providers-and-models.md) · [Contents](../../README.md#contents) · [.agent-mem.json](project-settings.md) →
