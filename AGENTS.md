# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## What This Is

Hermes HUD Web UI — a browser-based dashboard for monitoring the Hermes AI agent. It reads agent data from `~/.hermes/` and displays identity, memory, skills, sessions, cron jobs, projects, costs, activity patterns, corrections, sudo governance, and live chat across 19 tabs.

## Commands

### Development Setup (one-time)
```bash
./install.sh        # Builds frontend, installs Python package
```

### Full-Stack Dev
```bash
hermes-hudui --dev          # Terminal 1: backend on :3001 (auto-reload)
cd frontend && npm run dev  # Terminal 2: frontend on :5173 (proxies /api → :3001)
```

### Frontend
```bash
cd frontend
npm run dev      # Dev server on :5173
npm run build    # Production build (runs tsc first)
npm run lint     # ESLint
npm run preview  # Preview production build
```

### Backend CLI
```bash
hermes-hudui                         # Serve on :3001
hermes-hudui --port 8080             # Custom port
hermes-hudui --hermes-dir /path      # Override ~/.hermes/ location
hermes-hudui --host 0.0.0.0 --unsafe-allow-remote  # Trusted LAN only
```

### Release Workflow
```bash
# 1. Bump version in: pyproject.toml, App.tsx, BootScreen.tsx, CHANGELOG.md
# 2. Build + deploy static assets:
./scripts/build_frontend.sh
# 3. Commit, tag, push:
git add -f backend/static/assets/ && git commit && git tag v0.X.Y && git push --tags
# 4. GitHub release:
gh release create v0.X.Y --title "v0.X.Y" --notes "..."
```

## Architecture

```
React Frontend (Vite + Tailwind)
    ↓ /api/* (proxied in dev)
FastAPI Backend (Python)
    ↓ collectors/*.py        ↓ chat/engine.py
~/.hermes/ (agent data)     hermes CLI (subprocess)
```

### Backend (`backend/`)

- **`main.py`** — FastAPI app + CLI entry point. Sets `HERMES_HOME`, starts Uvicorn.
- **`collectors/`** — One module per data domain (memory, skills, sessions, cron, projects, patterns, sudo). Each reads `~/.hermes/` and returns dataclasses from `models.py`.
- **`models.py`** — All dataclasses (`HUDState`, `MemoryState`, `SkillsState`, etc.). `@property` fields are included in serialization.
- **`serialize.py`** — `to_dict()` recursively converts dataclasses to JSON-safe dicts.
- **`routes/`** — FastAPI route handlers that call collectors and return serialized data.
- **`api/memory.py`** — CRUD endpoints for memory editing. Uses `fcntl.flock` + atomic writes (`tempfile.mkstemp` → `os.replace`) matching hermes-agent's `MemoryStore` locking pattern.
- **`api/sessions.py`** — Session search (title + FTS). Filters `source != 'tool'` to exclude HUD-generated sessions.
- **`api/chat.py`** — Chat session CRUD, SSE streaming endpoint, cancel endpoint.
- **`chat/engine.py`** — Singleton `ChatEngine` spawning `hermes chat -q <msg> -Q --source tool` per message. Captures `hermes_session_id` from stdout, queries `state.db` post-completion for tool calls and reasoning.
- **`chat/streamer.py`** — SSE event emitter (`emit_token`, `emit_tool_start`, `emit_tool_end`, `emit_reasoning`, `emit_done`).
- **`cache.py`** — Mtime-based cache invalidation (sessions 30s, skills 60s, patterns 60s, profiles 45s). Endpoints: `GET /api/cache/stats`, `POST /api/cache/clear`.
- **`file_watcher.py`** — Watches `~/.hermes/` via native filesystem events and broadcasts `data_changed` events. Frontend auto-refreshes via SWR mutation.

### Frontend (`frontend/src/`)

- **`App.tsx`** — Root: tab manager, theme provider, command palette. Chat tab uses fixed-height container; other tabs scroll normally.
- **`hooks/useApi.ts`** — SWR wrapper with auto-refresh, 5s dedup, 3 retries.
- **`hooks/useChat.ts`** — Chat state: SSE streaming, session CRUD, per-session message cache (in-memory `Map` + localStorage persistence). Restores messages on session switch and page refresh.
- **`components/Panel.tsx`** — Shared panel wrapper (title, border, glow). Exports `CapacityBar`, `Sparkline`. `noPadding` prop for ChatPanel.
- **`components/chat/`** — `SessionSidebar`, `MessageThread`, `MessageBubble`, `Composer`, `ToolCallCard`, `ReasoningBlock`.
- **`components/MemoryPanel.tsx`** — Inline editing with hover-reveal controls, two-click delete, expandable add form.
- **`lib/utils.ts`** — `timeAgo()`, `formatDur()`, `formatTokens()`, `formatSize()`, `truncate()`.

## Key Conventions

**Adding a tab:** Create collector in `backend/collectors/`, dataclass in `models.py`, route in `backend/routes/`, panel component with `useApi`, register in `TopBar.tsx` TABS + `App.tsx` TabContent/GRID_CLASS.

**Chat engine:** Stateless per-message subprocess. No backend message persistence — history lives in localStorage. On server restart, ChatPanel re-creates backend sessions and migrates localStorage keys to new IDs.

**Memory editing:** Sync `def` endpoints (not `async`) so FastAPI auto-threads blocking I/O. File locking via `fcntl.flock` on `.lock` files. Atomic writes via `tempfile.mkstemp` + `os.replace`. Entries delimited by `\n§\n`.

**Styling:** Tailwind for layout, CSS variables (`var(--hud-*)`) for theming. Funnel Sans font. Five themes: `ai`, `hermes`, `blade-runner`, `fsociety`, `anime`.

**TypeScript:** Use `any` for API response types — schema owned by backend.

**Version strings:** Must stay in sync across `pyproject.toml`, `App.tsx` status bar, `BootScreen.tsx`, and `CHANGELOG.md`.

**Token costs:** Hardcoded `MODEL_PRICING` in `backend/api/token_costs.py`. Unknown models remain explicitly unpriced instead of inheriting a misleading rate.

**Sudo collector:** `backend/collectors/sudo.py` mines `state.db` tool-output messages via FTS for sudo command executions, parses `config.yaml` for approval/security settings, and tails `logs/gateway.log` for explicitly approved commands. Outcome classification: `exit_code=-1` + "approval" in error = blocked; password error in output = failed; `exit_code=0` = success.

**Shared YAML loader:** `backend/collectors/utils.py` exports `load_yaml(text)` — tries `yaml.safe_load`, falls back to a minimal line parser. Used by `config.py` and `sudo.py`.

## Cursor Cloud specific instructions

This estate is a multi-repo workspace under `/agent/repos/*`. The startup update script refreshes dependencies for the runnable/testable code repos; the notes below are for running and validating them. Standard commands live in each repo's README/AGENTS.md and `package.json`/`pyproject.toml` — this section only records the non-obvious Cloud caveats.

### Python setup uses `uv` (not `python3 -m venv`)
The base image has no `python3-venv`/`ensurepip`, so `python3 -m venv` fails. Use `uv` (installed by the update script to `~/.local/bin`; add it to `PATH`). Create/refresh a venv with `uv venv <dir> --python 3.12` and install with `uv pip install --python <dir>/bin/python ...`. Re-running `uv venv` on an existing venv is a harmless no-op (prints a `--clear` hint); `uv pip install` then refreshes it.

### e-Hermes-HUD-UI (flagship — this repo, full-stack web app)
- Run dev: `source venv/bin/activate && hermes-hudui --dev` (backend :3001, auto-reload) and, in `frontend/`, `npm run dev` (Vite :5173, proxies `/api` → :3001). See AGENTS.md "Commands".
- Vite binds to `localhost`/`::1`, **not** `127.0.0.1` — probe it with `curl http://localhost:5173`, not the dotted IP. `ss`/`netstat` are not installed; use `curl` to check ports.
- The dashboard reads `~/.hermes/` and is **empty on a fresh VM**. Seed `~/.hermes/config.yaml` and `~/.hermes/memories/{MEMORY.md,USER.md}` (entries `§`-delimited) to get meaningful data. The CHAT tab shells out to a `hermes` CLI that is not present in Cloud, so live chat will error — every other tab works from seeded files.
- Backend tests: `source venv/bin/activate && python -m pytest` (pytest is added by the update script). Frontend `npm run lint` emits 5 `react-refresh` warnings by design (0 errors).

### Other estate code repos (deps installed by the update script)
- `org-Context7` (pnpm): `pnpm build`, `pnpm lint:check` pass. `pnpm test` — the `sdk` package's network tests need `CONTEXT7_API_KEY`; unit tests pass without it. The MCP server runs via `node packages/mcp/dist/index.js` and needs outbound network + an API key to be useful.
- `e-CodeKB-Logic` (pnpm, Node ≥22, pnpm ≥10): `pnpm build`, `pnpm lint`, `pnpm test` all pass; `pnpm dev:dashboard` starts the Vite dashboard.
- `e-VSP-Protocol` (uv/`.venv`): `.venv/bin/pytest` passes; `vsp` is a CLI (no server). `ruff check .` currently reports pre-existing style findings in repo code.
- `e-Logis-Dashboard` (uv/`.venv`): needs a `.env` (copy `.env.example`; it is gitignored) before `bash scripts/start.sh` (uvicorn :8787). The UI shell + verify checks (`py_compile`, `node --check static/app.js`, `node tests/kill-voice.test.js`) run in Cloud; full Hermes (:8642) / Kokoro TTS (:8085) integration is laptop-only.

### Content-only repos (no dependency install needed)
`org-Skills`, `org-Social-Media`, `org-Taste` are agent-skill/prompt content. `org-Marketing`, `org-Superpowers`, `org-UI-UX-Pro-Max`, `Logistemia` are content plus light optional tooling. `e-Kokoro-Voice` (TTS) needs `espeak-ng` + a large Hugging Face model download; `G6Fx-Prime` is an incomplete umbrella skeleton (missing `pyproject.toml`, empty submodules) — neither is wired into the update script.
