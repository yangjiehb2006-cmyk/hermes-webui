# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hermes WebUI is a lightweight, dark-themed web interface for [Hermes Agent](https://hermes-agent.nousresearch.com/). It provides full parity with the CLI experience — chat, workspace file browser, cron/skills/memory viewers, profiles — in a browser.

**Tech stack:** Python stdlib server (ThreadingHTTPServer) + vanilla JS (no build step, no framework). The server uses only `pyyaml` as a dependency; all ML/agent deps live in the Hermes agent venv.

## Common Commands

### Starting the server

```bash
# Auto-discovers Hermes agent, Python, and state directories
./start.sh

# Or directly with Python (use the Hermes agent venv Python)
HERMES_WEBUI_PORT=8787 python server.py

# Docker
docker compose up -d
```

### Running tests

```bash
# All tests (uses isolated port 8788, separate state dir)
pytest tests/ -v --timeout=60

# Single test file
pytest tests/test_sprint1.py -v

# Single test
pytest tests/test_sprint1.py::test_name -v
```

### Health check
```bash
curl http://127.0.0.1:8787/health
```

## Architecture

### Backend (`api/`)

```
server.py          Entry point + HTTP Handler (~60 lines). Thin shell.
api/
  routes.py        All GET/POST handlers (~2000 lines)
  config.py        Discovery, globals, model detection, reloadable config
  models.py        Session model + CRUD
  streaming.py     SSE engine, run_agent, cancel support
  auth.py          Optional password authentication, signed cookies
  workspace.py     File ops: list_dir, read_file_content, git detection
  upload.py        Multipart parser, file upload handler
  onboarding.py    First-run wizard, provider setup
  profiles.py      Profile state management
  helpers.py       HTTP helpers: j(), bad(), require(), safe_resolve()
  state_sync.py   /insights sync
  updates.py       Self-update check
```

### Frontend (`static/`)

Six vanilla JS modules loaded in dependency order:
```
ui.js        DOM helpers, renderMd, tool cards, global state S
workspace.js File preview, file ops, git badge
sessions.js  Session CRUD, list rendering, search, projects
messages.js  send(), SSE handlers, approval, transcript
panels.js    Cron, skills, memory, workspace, profiles, settings, todos
boot.js      Event wiring, mobile nav, voice input, boot IIFE
style.css    All CSS including mobile responsive + 7 themes
index.html   HTML template
```

### Key patterns

**SSE Streaming:** Two-endpoint pattern:
1. `POST /api/chat/start` — creates queue.Queue, spawns daemon thread running `_run_agent_streaming()`, returns `{stream_id}`
2. `GET /api/chat/stream?stream_id=X` — long-lived SSE connection reading from the queue

**Session model:** Plain Python class (not dataclass). Sessions stored as JSON files in `~/.hermes/webui-mvp/sessions/`. In-memory cache with LRU eviction (OrderedDict, cap 100).

**Workspace convention:** Each user message is prefixed with `[Workspace: /absolute/path]` — this is the authoritative workspace source. Always resolve relative paths against this workspace.

### State directory (`~/.hermes/webui-mvp/`)

```
sessions/           One JSON file per session: {session_id}.json
workspaces.json     Registered workspaces list
last_workspace.txt  Last-used workspace path
settings.json       User settings (model, workspace, send key, password hash)
projects.json       Session project groups
```

## Critical Rules (Do Not Regress)

These have been broken and fixed multiple times:

1. **`/api/upload` must be checked BEFORE `read_body()` in `do_POST`.** `read_body()` consumes the request body stream. Upload parsing also needs the body. Order matters.

2. **`deleteSession()` must NEVER call `newSession()`.** Deleting does not create. If deleting active session with others remaining, load `sessions[0]`. If none remain, show empty state.

3. **`run_conversation()` takes `task_id=`, NOT `session_id=`.** This is a silent TypeError trap.

4. **`stream_delta_callback` receives `None` as end-of-stream sentinel.** Guard: `if text is None: return`

5. **`send()` must capture `activeSid` BEFORE any await.** Session can change mid-flight due to concurrent requests.

6. **All `SESSIONS` dict accesses must hold `LOCK`.** `with LOCK: ...`

7. **`require()` and `bad()` helpers for validation.** Never expose tracebacks to API clients — return clean 400/404 JSON.

8. **`pattern_keys` (plural), not `pattern_key` (singular).** The approval module may include both; always iterate `pattern_keys`.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `HERMES_WEBUI_AGENT_DIR` | auto-discovered | Path to hermes-agent checkout |
| `HERMES_WEBUI_PYTHON` | auto-discovered | Python executable |
| `HERMES_WEBUI_PORT` | `8787` | Port |
| `HERMES_WEBUI_HOST` | `127.0.0.1` | Bind address |
| `HERMES_WEBUI_STATE_DIR` | `~/.hermes/webui-mvp` | Sessions/state storage |
| `HERMES_WEBUI_DEFAULT_WORKSPACE` | `~/workspace` | Default workspace |
| `HERMES_WEBUI_DEFAULT_MODEL` | `openai/gpt-5.4-mini` | Default model |
| `HERMES_WEBUI_PASSWORD` | *(unset)* | Set to enable password auth |
| `HERMES_HOME` | `~/.hermes` | Base directory for Hermes state |

## Adding a New API Endpoint

**Backend** (in `api/routes.py`):
- GET: add before the 404 fallback in `handle_get`
- POST: add after `/api/upload` check and `read_body()`, before 404 fallback
- Always validate required fields with `require()` → return 400 on missing
- Use `get_session(sid)` with `try/except KeyError` → return 404

**Frontend** (in appropriate `static/` module):
```javascript
// GET
const data = await api('/api/your/endpoint?param=' + encodeURIComponent(value));

// POST
const data = await api('/api/your/endpoint', {
  method: 'POST',
  body: JSON.stringify({field: value})
});
```

## Key Files for Reference

| File | Purpose |
|---|---|
| `ARCHITECTURE.md` | Canonical architecture doc — endpoints, data flow, ADRs, sprint log |
| `AGENTS.md` | Workspace convention: `[Workspace: /path]` prefix handling |
| `TESTING.md` | Manual browser test plan + automated coverage reference |
| `api/streaming.py` | SSE engine, agent invocation, approval integration |
| `api/models.py` | Session model, SESSIONS cache, CRUD operations |
| `static/messages.js` | `send()` function, SSE event handlers, INFLIGHT tracking |
| `static/sessions.js` | `deleteSession()` rules, session list rendering |

## Known Technical Debt

- **Thread-safety (Phase B):** Env vars `TERMINAL_CWD`, `HERMES_EXEC_ASK`, `HERMES_SESSION_KEY` are process-global. Per-session lock added in Sprint 4. Full fix needs terminal tool to read thread-local.
- **renderMd() gaps:** Nested lists and mixed bold+link on same line may produce garbled output. Tables partially supported.
