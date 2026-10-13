# Measurements

Hook latency, retrieval quality on LongMemEval-S and on the repository's fixture, and how to run them yourself.

Every number on this page comes from a run on a real install. The date and the command are given, so you can run
it again.

## Hook latency

**2026-10-13**, agent-mem 1.0.0 (7d6cf39), Python 3.13, Apple Silicon Mac, a fresh data folder, 40 sessions of six
Claude Code events each. Times are the whole `agent-mem hook claude` process as the agent sees it:

| Event | Average | p50 | p95 | p99 |
| --- | --- | --- | --- | --- |
| SessionStart | 70 ms | 69 ms | 79 ms | 87 ms |
| UserPromptSubmit | 67 ms | 65 ms | 74 ms | 103 ms |
| PreToolUse | 65 ms | 64 ms | 72 ms | 93 ms |
| PostToolUse | 66 ms | 64 ms | 76 ms | 98 ms |
| Stop | 65 ms | 64 ms | 73 ms | 81 ms |
| SessionEnd | 64 ms | 63 ms | 72 ms | 82 ms |
| `python -c pass` | 29 ms | 29 ms | 30 ms | 31 ms |

About 29 ms of each is Python starting. The rest is the hook's own work, well inside its 400 ms deadline. A
`PreToolUse` with nothing to check (no rules, no file history) exits through the fast path about 45 ms sooner.

```python title="the script (shortened)"
for i in range(40):
    sid = f"bench-{i}"
    for name, payload in six_events(sid):           # SessionStart … SessionEnd, Claude Code payloads
        t = time.perf_counter()
        subprocess.run(["agent-mem", "hook", "claude"], input=json.dumps(payload),
                       capture_output=True, text=True,
                       env={**os.environ, "AGENT_MEM_DATA_DIR": tmp, "AGENT_MEM_NO_SPAWN": "1"})
        times[name].append((time.perf_counter() - t) * 1000)
```

## Retrieval: LongMemEval-S

[LongMemEval](https://arxiv.org/abs/2410.10813) asks questions about long chat histories. The `_s` set has 500
questions over histories of about 115k tokens each; agent-mem answers retrieval: does the right session appear in
the top 5 or 10? **470 questions** (the 30 abstention questions have no session to find).

**2026-10-12**, SQLite backend, MiniLM-L12-v2, no model call:

| Category | n | R@5 | R@10 | All@5 | All@10 |
| --- | --- | --- | --- | --- | --- |
| knowledge-update | 72 | 1.000 | 1.000 | 1.000 | 1.000 |
| multi-session | 121 | 0.975 | 0.992 | 0.777 | 0.876 |
| single-session-assistant | 56 | 1.000 | 1.000 | 1.000 | 1.000 |
| single-session-preference | 30 | 0.900 | 1.000 | 0.900 | 1.000 |
| single-session-user | 64 | 0.984 | 1.000 | 0.984 | 1.000 |
| temporal-reasoning | 127 | 0.961 | 0.984 | 0.787 | 0.874 |
| **overall** | **470** | **0.9745** | **0.9936** | **0.8766** | **0.934** |

R@k: at least one evidence session in the top k. All@k: every evidence session in the top k.

```sh
agent-mem eval longmemeval_s.json --progress lme.progress.jsonl
```

## Retrieval: the repository's fixture

`tests/fixtures/memory_cases.json`: 20 answerable questions, 20 that must get no answer, 10 pairs where a newer
memory replaces an older one. **2026-10-13**, fresh database, SQLite backend, MiniLM-L12-v2:

| Metric | Value | Meaning |
| --- | --- | --- |
| `hit@1` | 1.00 | the top raw search result contains the expected answer |
| `abstention` | 0.45 | raw search returns nothing for an unanswerable question (raw search always lists candidates) |
| `injected_hit@1` | 0.90 | what the prompt hook injects has the answer first |
| `injected_abstention` | **1.00** | the prompt hook injects nothing for every unanswerable question |
| `mcp_hit@1` | 1.00 | `mem_search`'s first confident result has the answer |
| `mcp_abstention` | **1.00** | `mem_search` says "no confident match" for every unanswerable question |
| `contradictions_resolved` | 0/10 | the top result shows only the newer value |

The last line is expected without a model: agent-mem keeps the old and the new fact, shows both with dates, and the
agent reads the newer one as current. The daily review, which needs a model, retires the old one. The fixture runs
without it.

```sh
agent-mem eval --fixture tests/fixtures/memory_cases.json
```

> [!NOTE]
> The abstention numbers are what matter most for injection: an agent given an unrelated memory is worse off than one
> given nothing. The hook and `mem_search` both abstained on all 20 unanswerable questions.

## Load

**2026-10-13**, agent-mem 1.0.0 (7d6cf39), a fresh data folder with the `fake` model provider. Six sessions of ten
requests each fired real hook processes in parallel, rotating Claude Code, Codex and Copilot payloads, while a
second process took the database's write lock for 800 ms every 1.5 seconds, and the indexer, the observer, the
viewer and MCP search ran alongside:

| Check | Result |
| --- | --- |
| hook processes | 612 in 7.3 s |
| hook latency | median 67 ms, p95 101 ms, max 272 ms; none over the 400 ms deadline |
| invalid hook output or stderr | none |
| viewer and search errors during the run | none |
| requests complete, every tool call in its request, spool drained, nothing quarantined | yes |
| observer queue afterwards | 60 of 60 done |
| health errors | none |

Fewer `pre_tool` events (152) than tool calls (240) is the [fast path](architecture/hooks.md#pretooluse-warnings-blocks-and-file-history):
calls with nothing to check are not stored.

## Run them yourself

| What | Command |
| --- | --- |
| fixture | `agent-mem eval --fixture tests/fixtures/memory_cases.json` (from a checkout) |
| LongMemEval-S | download `longmemeval_s.json`, then `agent-mem eval longmemeval_s.json` |
| compare with mem0's own search | `agent-mem eval longmemeval_s.json --backend mem0` |
| without vectors | add `--no-semantic` |
| hook latency | the script above with your own payloads |

`agent-mem eval` builds its own database in a temporary folder; it never touches your memory.

---

← [Troubleshooting](troubleshooting.md) · [Contents](../README.md#contents) · [Development](development.md) →
