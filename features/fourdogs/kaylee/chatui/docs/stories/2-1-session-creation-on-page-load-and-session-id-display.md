# Story 2.1: Session creation on page load and session ID display

Status: ready-for-dev

## Story

As a developer,
I want a Kaylee session automatically created when I open the UI,
so that I can immediately start testing without any manual setup or configuration.

## Acceptance Criteria

1. On page load, `POST /sessions` is called before any user interaction
2. The returned `session_id` is displayed in the page header (e.g. `Session: abc-123`)
3. If `POST /sessions` fails (network error or 5xx), an inline error banner is displayed
4. Reloading the page creates a new session (prior session abandoned — expected)
5. After session creation succeeds, `GET /sessions/{id}/messages` is called to load history

## Tasks / Subtasks

- [ ] Replace `ui/index.html` placeholder with full HTML structure (AC: 1, 2, 3, 5)
  - [ ] Header bar: title + session ID display span
  - [ ] Conversation area: scrollable div for messages
  - [ ] Input bar: text input + Send button + status line
  - [ ] `<script>` block with all JS inline (NFR-04 — single file)
- [ ] Implement `createSession()` async function in JS (AC: 1, 2, 3)
  - [ ] `POST /sessions` with `Content-Type: application/json`
  - [ ] On success: store `sessionId` in module-level `let sessionId = null`
  - [ ] On success: update header span with session ID
  - [ ] On failure: display error banner (see error handling pattern below)
- [ ] Implement `loadHistory()` async function — called after `createSession()` resolves (AC: 5)
  - [ ] `GET /sessions/{sessionId}/messages`
  - [ ] On success: pass messages array to `renderMessages()` (stub for Story 2.3)
  - [ ] On empty array: no-op (graceful empty state)
- [ ] Call `createSession().then(loadHistory)` on `DOMContentLoaded` (AC: 1, 5)

## Dev Notes

- **Session ID storage:** `let sessionId = null` at top of script block — module-level, no `localStorage`
- **Error banner pattern:** `<div id="error-banner" style="display:none">` — show/hide inline, don't crash
- **API base URL:** Use relative path `/sessions` — same-origin, no hardcoded host (TDL-003)
- **POST /sessions response shape:** `{ "session_id": "uuid-string" }`
- **GET /sessions/{id}/messages response shape:** `[{ "role": "user"|"assistant", "content": "..." }]`
- **Call order is mandatory:** `POST /sessions` must succeed before `GET /sessions/{id}/messages` fires. Do not fire history load if session creation failed.
- **No framework — use `fetch()`** — not XHR, not axios, not jQuery (NFR-01)

### Project Structure Notes

- All JS and CSS remain inline in `ui/index.html` (NFR-04 — single file constraint)
- No `app.js`, no `style.css`, no `node_modules`
- `ui/` directory already created in Story 1.1

### References

- Architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — Session Initialization flow, D3, D5
- Tech Decisions: `docs/fourdogs/kaylee/chatui/tech-decisions.md` — TDL-003, TDL-005
- PRD: `docs/fourdogs/kaylee/chatui/prd.md` — FR-02, FR-03
- Adversarial review note: history load on new session returns empty array — handle gracefully, not as error

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `ui/index.html` (modified — full HTML structure replaces placeholder)
