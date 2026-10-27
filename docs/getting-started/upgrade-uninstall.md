# Upgrade & uninstall

Move to the newest version without a gap, or remove agent-mem cleanly.

## Upgrade

```sh
agent-mem upgrade
```
1. Installs the newest commit of the source you installed from, in place, with the extras you have
   (`uv tool install --upgrade "agent-mem[…] @ git+https://github.com/Abuarchiv/agent-mem"`). A branch or tag you
   named stays.
2. Downloads a model the new release needs (`agent-mem models install`).
3. Runs `agent-mem doctor --fix` with the new code: migrates the database and rewrites hooks that an older release
   wrote, at the same binary path.
```text
agent-mem 1.0.0 (7d6cf39) -> 1.0.0 (86ad40c). Hooks use it now; running sessions keep their MCP server until restart.
```

Add an extra at the same time: `agent-mem upgrade --extras observer`. Running the install script again does the same
as `upgrade`.

### On Windows

A running program cannot be replaced on Windows, and the viewer, background jobs and every open session's MCP
server run from agent-mem's own folder. So `agent-mem upgrade` hands over to a PowerShell script and ends; the
script waits for it, stops those processes, installs, restarts the viewer if it was running, downloads models and
runs `doctor --fix`. `logs\upgrade.log` in the data folder shows how it went. Restart open sessions afterwards so
they get their search back. When the installed commit is already the newest, nothing is stopped.

### Docker

`agent-mem upgrade` builds the new image and replaces the container; memory and logins stay.

### While sessions run

You do not need to close your agents.

- Hooks use the new code from the next event. A hook that meets the old schema spools its event and the indexer
  migrates, then replays.
- The viewer restarts itself on its next page load ("agent-mem was updated").
- An MCP server keeps running the old code until the agent restarts it.
- **Codex** asks you to trust changed hooks once more in `/hooks`.

> [!WARNING]
> Do not use `uv tool install --reinstall`. It deletes the environment before rebuilding it, and hooks firing in those
> seconds find no binary. `agent-mem upgrade` swaps the environment in place.

## Uninstall

Order matters: hooks that point at a removed binary fail on every event.
1. Remove hooks, MCP entries and the viewer's login item:

   ```sh
   agent-mem uninstall all
   ```

2. Delete the data, if you want to:

   ```sh
   agent-mem purge --all --yes
   ```

3. Remove the command:

   ```sh
   uv tool uninstall agent-mem
   ```
`uninstall` removes exactly what `setup --write` wrote; your own hooks and settings stay. `purge --all` deletes the
database, spool, quarantine and logs and keeps `config.json` and the downloaded models. To remove those as well,
delete the data folder (`agent-mem paths`).

If you installed the Claude Code plugin, also run `/plugin uninstall agent-mem@agent-mem`.

### Already removed the command first?

The login item removes itself once the interpreter is gone (launchd and the Windows Startup file delete
themselves; the systemd unit is skipped). The hooks stay in the agents' config files: remove the entries that
contain `agent-mem hook` from `~/.claude/settings.json` and `~/.codex/hooks.json`, delete
`~/.copilot/hooks/agent-mem.json` and `~/.config/opencode/plugins/agent-mem.ts`, or install agent-mem again and
run `agent-mem uninstall all`.

---

← [Your first session](first-session.md) · [Contents](../../README.md#contents) · [Claude Code](../agents/claude-code.md) →
