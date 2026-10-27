# Development

Working on agent-mem itself, and working on these docs.

## agent-mem

```sh
git clone https://github.com/Abuarchiv/agent-mem
cd agent-mem
uv sync --all-extras
uv run pytest -m "not local"
uv run ruff check src tests && uv run ruff format --check src tests && uv run pyright
```

CI runs lint, types, tests and a wheel smoke test on Ubuntu, macOS (arm64 and Intel) and Windows.

### Rules the code keeps

- **The hook path imports the standard library only.** `core/`, `hooks/` and the lexical half of `memory/` never
  import mem0, Qdrant, fastembed or numpy; a test enforces it.
- **Dependencies point one way:** `core` ← `hooks`, `memory` ← `observe`, `mcp`, `ui`, `bench` ← `cli`.
- **A hook never blocks, never loses an event, never prints invalid output.** Exit 0 always; spool on doubt.
- **SQLite is the truth.** Anything derived (mem0's index) must be rebuildable from it.
- **One write path per kind of data.** Events go through `capture.apply`, memories through `store.add_memory`,
  retiring through `store.supersede`. Fix a bug where all callers pass, not in one caller.
- **Bug fixes need a test that fails before the change.**
- **Text for people is plain.** The fixed words are request, agent, recording and failed attempt; the
  technical terms stay in text only the model reads.

### Running against a scratch database

```sh
export AGENT_MEM_DATA_DIR=/tmp/am-dev AGENT_MEM_NO_SPAWN=1
python3 -c 'import json; print(json.dumps({"session_id": "dev", "cwd": "'"$PWD"'",
  "hook_event_name": "UserPromptSubmit", "prompt": "run the tests"}))' | uv run agent-mem hook claude
uv run agent-mem status
```

`AGENT_MEM_FAKE_LLM=answers.json` with `"observe": {"providers": ["fake"]}` runs the observer against canned answers.

### Adding an agent

See [Hook Lifecycle: Adding another agent](architecture/hook-lifecycle.md#adding-another-agent).

## These docs

The site is [Astro Starlight](https://starlight.astro.build/) in TypeScript, styled like the agent-mem viewer.
Source: [github.com/Abuarchiv/agent-mem-docs](https://github.com/Abuarchiv/agent-mem-docs).

```sh
npm install
npm run dev          # http://localhost:4321/agent-mem-docs/
npm run build
npm run diagrams     # re-render every diagram
AGENT_MEM_SRC=../agent-mem npm run check   # the coverage check against an agent-mem checkout
npm run markdown -- ../agent-mem-md         # the same pages as plain Markdown
```

### Diagrams

Diagrams are [Excalidraw](https://excalidraw.com) scenes, written as small TypeScript files in `diagrams/` with the
viewer's palette. `npm run diagrams` turns each into Excalidraw elements and exports them with Excalidraw's own SVG
renderer in headless Chromium:

| File | What |
| --- | --- |
| `diagrams/<name>.ts` | the scene: boxes, arrows, sequence diagrams (`diagrams/_lib.ts`) |
| `public/diagrams/<name>.excalidraw` | the scene for excalidraw.com: open it there to edit by hand |
| `public/diagrams/<name>.svg` | what the page shows |

Every diagram on the site has an "Open in Excalidraw" link to its scene file.

### Plain Markdown

`scripts/export-markdown.ts` writes the same pages as GitHub Markdown for
[agent-mem-docs-md](https://github.com/Abuarchiv/agent-mem-docs-md): asides become alerts, tabs bold labels, cards
lists, diagrams an image with a link to their Excalidraw scene, and every page gets previous/next links.

### The coverage check

`scripts/check-coverage.ts` reads an agent-mem checkout and fails when the docs miss something the code has: a
table, a config key, a CLI command or option, an environment variable, an MCP tool, or a hook event of any agent.
It also compares every config default with the "all defaults" block and the tables.
CI runs it against the newest agent-mem commit on every push and every day.

```sh
AGENT_MEM_SRC=../agent-mem npx tsx scripts/check-coverage.ts
```

> [!TIP]
> Payloads, outputs and timings on these pages were captured from real hook runs against a scratch data folder; the
> scripts are in `scripts/`.

---

← [Measurements](measurements.md) · [Contents](../README.md#contents)
