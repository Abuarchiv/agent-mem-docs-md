# .agent-mem.json

Per-project settings in the repository root. Only two harmless keys are honoured.

A file named `.agent-mem.json` in a project's root changes how agent-mem treats that one project. It can be
committed, so everyone who uses agent-mem in the repository gets the same behaviour.

```json title=".agent-mem.json"
{
  "capture": false,
  "exclude": ["fixtures/customers/**", "*.sqlite"]
}
```

| Key | Type | Effect |
| --- | --- | --- |
| `capture` | bool | `false`: nothing from this project is recorded; the project is not even registered |
| `exclude` | list of globs (at most 100) | added to the global `exclude_globs` for this project: content of matching files is never stored |

Nothing else is read. Budgets, models, providers and every other setting can only be changed in the user's own
[`config.json`](config-json.md). A repository you clone must not be able to switch on blocking rules,
point the observer at another model or raise limits on your machine.

## Where the root is

The project root is the git repository root (the `git rev-parse --show-toplevel` of the working folder), or the
folder itself outside git. A nested repository has its own root and its own `.agent-mem.json`. See
[how a folder becomes a project](../architecture/capture-and-privacy.md#how-a-folder-becomes-a-project).

## Other ways to keep a project out

| Way | Scope |
| --- | --- |
| `.agent-mem.json` `{"capture": false}` | this repository, for everyone using agent-mem in it |
| `"excluded_projects": ["/path/to/repo"]` in `config.json` | this path and everything below it, on your machine |
| `agent-mem pause [--for 2h]` | everything, for a while |
| `<private>…</private>` in a prompt or file | that text |
| `AGENT_MEM_DISABLE=1` in the agent's environment | every hook of that agent process |

Already recorded data stays until you delete it: `agent-mem purge --project <root>`.

---

← [config.json](config-json.md) · [Contents](../../README.md#contents) · [CLI](cli.md) →
