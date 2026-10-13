# Token Accounting

How agent-mem measures what finding something out cost the agent, and what the "savings" line does and does not claim.

agent-mem compares two numbers: what a memory costs the agent to **read**, and the work it took the agent to
**find it out** in the first place. The first is estimated from the text. The second is measured from the agent's
own transcript.

## What one turn cost

After a turn closes, the indexer reads the agent's transcript and stores the **fresh** tokens of the model calls
between the turn's prompt and its end in `turns.work_tokens`:

| Agent | Transcript | Fresh tokens per call |
| --- | --- | --- |
| Claude Code | `~/.claude/projects/<folder>/<session>.jsonl` plus `<session>/subagents/*.jsonl` | `input_tokens + cache_creation_input_tokens + output_tokens` of each assistant message (a message streamed over several lines counts once) |
| Codex | `~/.codex/sessions/YYYY/MM/DD/*<session>.jsonl` (the day before and after too) | `last_token_usage.input_tokens − cached_input_tokens + output_tokens` of each `token_count` event whose running total grew |
| Copilot CLI | `~/.copilot/session-state/<session>/events.jsonl` | `inputTokens − cacheReadTokens + outputTokens` of each `session.usage_record` |
| OpenCode | — | not measured |

**Cache reads are left out.** Every step of an agent re-reads the whole context; counted, a 30-step turn would look
like millions of tokens. What is left is what the turn added: new input, cache writes and output, subagents included.

- A turn without any model call in its window gets `0` (measured, nothing spent). `NULL` means not measured.
- A turn closed late (stale, or by the next prompt) ends where the next turn starts, so it never takes the next
  turn's calls.
- Only sessions active in the last 30 days are measured; Claude Code deletes transcripts after 30 days by default.
- Each transcript is read from where the earliest still-open turn began (`meta work.offset:<path>`), so a large
  transcript is not re-read after every turn. A file shorter than the stored offset was replaced and is read again.
- `CLAUDE_CONFIG_DIR`, `CODEX_HOME` and `COPILOT_HOME` are followed.

## What a memory cost

```text
memory's share = its turn's work_tokens ÷ the memories that turn produced (native notes not counted)
```

A block that names several memories of one turn counts that turn once. A block that names the turn itself (`T5`)
counts the turn whole, and its memories add nothing more.

## What reading costs

The same estimate the budgets use: a quarter token per ASCII character, one token per other character.

## Where it shows

| Place | Text |
| --- | --- |
| terminal panel at session start | `Your savings: ~84,760 tokens (98% reduction from reuse)`, only when measured work stands behind the loaded entries and it exceeds their reading cost |
| `agent-mem status` | `Given to agents: today ~6,100 tokens · last 7 days ~6,100 tokens in 71 reminders (~139 per session)`, plus the same comparison for 7 days |
| viewer | a Tokens tile on Overview, a line on Health, `Savings` in each memory's detail |

Every injected block adds to `meta injected:<day>`: blocks, estimated tokens and measured work behind them.

```text
savings = work behind the block − tokens to read the block
percent = savings ÷ work behind the block
```

> [!WARNING]
> This is a comparison, not a measured saving. Nobody knows how much of that work the agent would have done again
> without the memory. Maybe all of it, maybe none. The number says what the knowledge cost to get, against what it
> costs to hand over.

---

← [Capture & Privacy](capture-and-privacy.md) · [Contents](../../README.md#contents) · [Troubleshooting](../troubleshooting.md) →
