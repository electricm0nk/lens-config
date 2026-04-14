# Architecture Decision Document — fourdogs-kaylee-chatui

**Initiative:** fourdogs-kaylee-chatui
**Track:** tech-change
**Phase:** TechPlan
**Date:** 2026-04-13
**Status:** Draft

---

## Purpose

A minimal standalone chat UI for testing and troubleshooting the kaylee-agent service directly — outside the fourdogs-central app. Enables iterating on Kaylee's AI behaviour, session management, and response quality without requiring a full fourdogs-central deployment.

This is a **developer test tool**, not a user-facing product. Scope is intentionally narrow.

---

## System Context

```
┌─────────────────────────────────────────────────────┐
│  Docker Compose / k8s pod (fourdogs-kaylee namespace) │
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │  kaylee-agent (FastAPI, port 8000)          │     │
│  │                                             │     │
│  │  GET  /ui          → serve ui/index.html    │     │
│  │  GET  /ui/static/* → serve ui/static/*      │     │
│  │                                             │     │
│  │  POST /sessions                             │     │
│  │  POST /sessions/{id}/messages               │     │
│  │  POST /sessions/{id}/stream (501 stub)      │     │
│  │  GET  /sessions/{id}/messages               │     │
│  │  DELETE /sessions/{id}                      │     │
│  └─────────────────────────────────────────────┘     │
│                                                       │
└─────────────────────────────────────────────────────┘
         ▲
         │  browser (developer workstation)
         │  http://localhost:8000/ui
```

No new containers. No new ports. No proxy. The UI is served directly by the existing FastAPI app from a `ui/` folder mounted via `StaticFiles`.

---

## Decisions

### D1 — Vanilla HTML/JS, zero build step

No bundler, no framework, no package.json. A single `index.html` with inline `<script>` and `<style>`. Rationale: zero toolchain friction for a debug tool. Any developer can read and modify it in one file without context.

### D2 — FastAPI StaticFiles mount

FastAPI's `StaticFiles` serves the `ui/` directory. One addition to `app/main.py`:

```python
from fastapi.staticfiles import StaticFiles
app.mount("/ui", StaticFiles(directory="ui", html=True), name="ui")
```

The UI is accessible at `http://localhost:8000/ui`. No nginx, no separate server process.

### D3 — Direct API calls (no proxy)

The browser calls kaylee-agent's REST endpoints directly (`/sessions`, `/sessions/{id}/messages`). No CORS issues since both UI and API are on the same origin (`localhost:8000`).

### D4 — Future-proof streaming display

The UI renders synchronous responses now (wrapping the full reply as a single chunk). The display layer is written to handle both:
- **Sync mode**: full response appears at once
- **Stream mode**: tokens append progressively via `EventSource`

A `KAYLEE_STREAM_ENABLED` constant (defaulting to `false`) controls which path is taken. When kaylee-agent implements `POST /sessions/{id}/stream`, flip to `true` — no architectural change needed.

### D5 — Session lifecycle managed by UI

The UI creates a new session on load (`POST /sessions`) and holds `session_id` in memory. No persistence needed — this is a test tool. Session is abandoned on page refresh (Kaylee's DB cleans up via TTL).

---

## Component Design

### File Structure (inside kaylee-agent repo)

```
kaylee-agent/
├── app/
│   └── main.py              ← add StaticFiles mount here
├── ui/
│   └── index.html           ← entire UI in one file
└── ...
```

### `ui/index.html` — Responsibilities

| Concern | Approach |
|---------|----------|
| Session creation | `POST /sessions` on page load; store `session_id` in memory |
| Send message | `POST /sessions/{id}/messages` with `{"message": text}` |
| Display response (sync) | Append full response as assistant bubble |
| Display response (stream-ready) | `EventSource` on `POST /sessions/{id}/stream` when enabled |
| Conversation history | Maintained by kaylee-agent; re-fetched via `GET /sessions/{id}/messages` on load |
| Error display | Inline error banner (HTTP 4xx/5xx shown clearly) |
| Status indicator | Sending / awaiting / error state on the input area |

### UI Layout

```
┌──────────────────────────────────────┐
│  🤖 Kaylee Test UI    [session: abc] │  ← header bar
├──────────────────────────────────────┤
│                                      │
│  [user]   What were top sellers?     │
│  [kaylee] Based on available data... │
│  [user]   Can you break that down?   │
│  [kaylee] Sure — here are the...     │
│                                      │  ← scrollable message area
│                                      │
├──────────────────────────────────────┤
│  [ Ask Kaylee...          ] [Send]   │  ← input bar
│    ● ready                           │  ← status line
└──────────────────────────────────────┘
```

---

## API Interaction

### Session Initialization

```
Page load
  → POST /sessions
  → { "session_id": "uuid" }
  → store session_id, display in header
  → (optional) GET /sessions/{id}/messages
    → render any prior history
```

### Send Message (Sync — current)

```
User submits message
  → disable input, set status "sending..."
  → POST /sessions/{session_id}/messages
      body: { "message": "user text" }
  → 200: { "session_id": "...", "message_id": "...", "response": "..." }
    → append user bubble + kaylee bubble
    → re-enable input, set status "ready"
  → 4xx/5xx:
    → show error banner, re-enable input
```

### Send Message (Stream — future, behind KAYLEE_STREAM_ENABLED flag)

```
User submits message
  → disable input, set status "streaming..."
  → POST /sessions/{session_id}/stream
      body: { "message": "user text" }
  → EventSource opens, tokens append to kaylee bubble progressively
  → event: done → close EventSource, re-enable input
  → event: error → show error banner, re-enable input
```

---

## Deployment

### Development (docker-compose)

No changes to docker-compose needed beyond ensuring the `ui/` folder is present in the image. Since the Dockerfile copies the app directory, `ui/` is included automatically if placed at the repo root.

### k8s / Production

This UI is a **dev tool only**. It should be:
- Excluded from production images via `.dockerignore` (or conditional `COPY`)
- Or gated behind a `KAYLEE_DEV_UI_ENABLED=true` env var in `main.py`

Recommended: env var gate so the same image can be used in both dev and prod without rebuilding.

```python
if os.getenv("KAYLEE_DEV_UI_ENABLED", "false").lower() == "true":
    app.mount("/ui", StaticFiles(directory="ui", html=True), name="ui")
```

---

## Out of Scope

- Auth / session cookies — the test UI is unauthenticated; assumes local dev access
- Multi-user support — single session per browser tab
- Persistence of sessions across reloads
- Tool call visualization (nice to have, not now)
- Theming / accessibility
- Mobile layout

---

## Implementation Notes

- `ui/` folder: committed to kaylee-agent repo on the `fourdogs-kaylee-chatui` feature branch
- `app/main.py` modification: minimal — 2 lines (import + mount)
- No new Python dependencies
- No changes to existing API endpoints
- No changes to existing tests
