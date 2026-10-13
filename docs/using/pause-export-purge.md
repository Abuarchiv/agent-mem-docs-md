# Pause, export, purge

Stop recording for a while, take your memory with you, or delete it for good.

## Pause

```sh
agent-mem pause --for 2h     # 30m, 2h, 1d …
agent-mem pause              # until resume
agent-mem resume
```

While paused nothing is recorded in any project. Recall keeps working: briefings, matches and MCP tools still read
the memory. The session panel says `Recording paused until 00:13. agent-mem resume turns it back on.`

## Export

```sh
agent-mem export --output memory.json
agent-mem export --project ~/code/calc --output calc.json
```

One JSON file, readable only by you:

```json
{
  "version": 2,
  "exported_at": "2026-10-13T20:13:06.157Z",
  "projects": [ … ],
  "sessions": [ … ],
  "turns":    [ {"id": 1, "prompt": "…", "answer": "…", "outcome": "ok", "files_json": "[\"calc.py\"]", … } ],
  "memories": [ {"id": 2, "kind": "decision", "title": "…", "body": "…", "trust": "agent", "invalid_at": null, … } ],
  "recipes":  [ … ],
  "rules":    [ … ],
  "profile":  [ … ]
}
```

Retired memories are included with their `invalid_at` and `superseded_by`. Tool events are not.

## Purge

Deletes for good: from the database, the search index, the vectors, the spool and quarantine, and (on the next
observer run) the fact index.

| Command | Deletes |
| --- | --- |
| `agent-mem purge --id T12` | one request, its events, the facts and lessons drawn from it, and its session's summary |
| `agent-mem purge --id M3` | one memory |
| `agent-mem purge --project ~/code/calc` | everything of that project (give its root folder) |
| `agent-mem purge --before 2026-01-31` | everything older than the date |
| `agent-mem purge --all` | the database, spool, quarantine and logs; `config.json` and models stay |

Add `--yes` in scripts; a terminal asks first, a script without `--yes` is refused:

```text
$ agent-mem purge --all
Refusing to delete ALL agent-mem data without --yes.
```

After a purge the database is vacuumed so the deleted text is not left in free pages. An offline snapshot page
written by `agent-mem view --output` is discarded too.

> [!NOTE]
> Agents can delete single entries of the current project with `mem_forget`. They cannot delete another project's
> memories; for those the tool points to `agent-mem purge --id`.

---

← [Privacy](privacy.md) · [Contents](../../README.md#contents) · [How facts are written](../facts/how-facts-are-written.md) →
