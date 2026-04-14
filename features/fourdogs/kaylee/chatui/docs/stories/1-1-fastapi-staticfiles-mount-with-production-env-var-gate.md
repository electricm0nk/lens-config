# Story 1.1: FastAPI StaticFiles mount with production env var gate

Status: ready-for-dev

## Story

As a developer,
I want the chat UI served at `localhost:8000/ui` by kaylee-agent itself,
so that I can access the test interface without running any additional services, containers, or toolchain.

## Acceptance Criteria

1. When `KAYLEE_DEV_UI_ENABLED=true` is set — navigating to `http://localhost:8000/ui` loads `ui/index.html`
2. When `KAYLEE_DEV_UI_ENABLED` is unset or `false` — navigating to `http://localhost:8000/ui` returns 404
3. No new Python dependencies are added by this change
4. `ui/index.html` renders without browser console errors when served via the `/ui` route

## Tasks / Subtasks

- [ ] Create `ui/` directory at kaylee-agent repo root, add `ui/index.html` placeholder (AC: 1, 4)
  - [ ] Placeholder content: `<h1>Kaylee Test UI</h1>` — minimal valid HTML
- [ ] Add `StaticFiles` mount to `app/main.py` behind env var gate (AC: 1, 2, 3)
  - [ ] Import: `from fastapi.staticfiles import StaticFiles` and `import os`
  - [ ] Conditional: `if os.getenv("KAYLEE_DEV_UI_ENABLED", "false").lower() == "true":`
  - [ ] Mount: `app.mount("/ui", StaticFiles(directory="ui", html=True), name="ui")`
- [ ] Add `KAYLEE_DEV_UI_ENABLED=true` to local dev env config (docker-compose or `.env`) (AC: 1)
- [ ] Verify `StaticFiles` is part of existing starlette/fastapi dependency — no new package needed (AC: 3)

## Dev Notes

- `StaticFiles` is part of `starlette` which is a required transitive dep of `fastapi` — verify in requirements.txt/pyproject.toml before proceeding
- Mount point MUST be `/ui` not `/` to avoid conflicting with API routes
- `html=True` enables automatic `index.html` serving for directory requests (so `/ui` and `/ui/` both work)
- Mount line goes AFTER all API route registrations in `main.py` — FastAPI evaluates routes top-to-bottom
- The env var check uses `"false"` as default (fail-closed for production safety — TDL-006)

### Project Structure Notes

- `ui/index.html` → kaylee-agent repo root, new `ui/` subdirectory
- `app/main.py` → add 3 lines (import os, import StaticFiles, conditional mount)
- No other files modified

### References

- Architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — D2 (FastAPI StaticFiles), D6 (env var gate)
- Tech Decisions: `docs/fourdogs/kaylee/chatui/tech-decisions.md` — TDL-002, TDL-006
- PRD: `docs/fourdogs/kaylee/chatui/prd.md` — FR-01, NFR-02, NFR-03

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `ui/index.html` (new)
- `app/main.py` (modified — 3 lines added)
