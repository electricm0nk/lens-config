---
initiative: chatui
track: tech-change
phase: devproposal
created: 2026-04-13
note: "Generated retroactively — tech-change track skipped businessplan phase. Captures requirements before devproposal."
---

# PRD — fourdogs-kaylee-chatui

## Overview

A developer test chat UI for kaylee-agent, served directly by the FastAPI app as static files. Enables direct conversation testing without requiring fourdogs-central.

## User

**Primary:** Kaylee backend developer testing AI session behaviour, prompt quality, and response format.

## Requirements

### Functional

| ID | Requirement |
|----|-------------|
| FR-01 | UI is accessible at `http://localhost:8000/ui` when `KAYLEE_DEV_UI_ENABLED=true` |
| FR-02 | Automatically creates a new Kaylee session on page load (`POST /sessions`) |
| FR-03 | Displays session ID in the UI header |
| FR-04 | User can type a message and send it (`POST /sessions/{id}/messages`) |
| FR-05 | Response is displayed in a chat bubble format (user and assistant turns) |
| FR-06 | Conversation history is loaded on page open (`GET /sessions/{id}/messages`) |
| FR-07 | Sending state is visible (disabled input + status indicator while awaiting response) |
| FR-08 | Error responses (4xx/5xx) are displayed inline without crashing |
| FR-09 | When streaming is enabled (flag), use `POST /sessions/{id}/stream` with `EventSource` |

### Non-Functional

| ID | Requirement |
|----|-------------|
| NFR-01 | Zero build toolchain — vanilla HTML/JS, no npm, no bundler |
| NFR-02 | Zero new dependencies — no new Python packages |
| NFR-03 | Production safety — mount only when `KAYLEE_DEV_UI_ENABLED=true` |
| NFR-04 | Single file — entire UI in `ui/index.html` |

## API Surface Used

| Endpoint | Purpose |
|----------|---------|
| `POST /sessions` | Create session on load |
| `POST /sessions/{id}/messages` | Send message (sync) |
| `GET /sessions/{id}/messages` | Load history |
| `POST /sessions/{id}/stream` | Future streaming (behind flag, currently 501) |

## Constraints

- Must run inside the kaylee-agent container (no separate service)
- Must share origin with the API (no CORS concerns)
- Dev-tool only — not for production user access

## Out of Scope

- Authentication
- Persistent sessions across browser refreshes
- Multi-tab or multi-user support
- Mobile/responsive layout
- Tool call visualization
