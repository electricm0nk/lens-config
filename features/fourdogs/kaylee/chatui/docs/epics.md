---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics, step-03-create-stories]
inputDocuments:
  - docs/fourdogs/kaylee/chatui/prd.md
  - docs/fourdogs/kaylee/chatui/architecture.md
  - docs/fourdogs/kaylee/chatui/tech-decisions.md
---

# fourdogs-kaylee-chatui — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for `fourdogs-kaylee-chatui`, decomposing the requirements from the PRD and Architecture into implementable stories. The initiative delivers a developer test chat UI for kaylee-agent, served as static files by the existing FastAPI app.

---

## Requirements Inventory

### Functional Requirements

- FR-01: UI accessible at `localhost:8000/ui` when `KAYLEE_DEV_UI_ENABLED=true`
- FR-02: Auto-create Kaylee session on page load (`POST /sessions`)
- FR-03: Display session ID in header
- FR-04: User can type and send a message (`POST /sessions/{id}/messages`)
- FR-05: Response displayed in chat bubble format (user and assistant turns)
- FR-06: Conversation history loaded on page open (`GET /sessions/{id}/messages`)
- FR-07: Sending state visible (disabled input + status indicator while awaiting response)
- FR-08: Error responses (4xx/5xx) displayed inline without crashing
- FR-09: When streaming flag = true, use `POST /sessions/{id}/stream` with `EventSource`

### Non-Functional Requirements

- NFR-01: Zero build toolchain — vanilla HTML/JS, no npm, no bundler
- NFR-02: Zero new Python dependencies
- NFR-03: Production safety — `StaticFiles` mount only when `KAYLEE_DEV_UI_ENABLED=true`
- NFR-04: Single file — entire UI in `ui/index.html`

### Additional Requirements

- FastAPI `StaticFiles` mount at `/ui` in `app/main.py` (D2 from architecture)
- `ui/` folder at kaylee-agent repo root
- `KAYLEE_STREAM_ENABLED` JS constant (default `false`) controls sync vs. EventSource path
- Session held in memory — no `localStorage`, no cookies (D5 from architecture)
- Graceful empty state when `GET /sessions/{id}/messages` returns empty array (adversarial review note)
- No additional containers or ports beyond existing kaylee-agent service

---

### FR Coverage Map

| FR | Epic | Note |
|----|------|------|
| FR-01 | Epic 1 | FastAPI mount + env var gate |
| NFR-02 | Epic 1 | No new Python deps |
| NFR-03 | Epic 1 | KAYLEE_DEV_UI_ENABLED=true gate |
| FR-02 | Epic 2 | POST /sessions on load |
| FR-03 | Epic 2 | Session ID in header |
| FR-04 | Epic 2 | POST /sessions/{id}/messages |
| FR-05 | Epic 2 | Chat bubble display (user + assistant) |
| FR-06 | Epic 2 | GET /sessions/{id}/messages history load |
| FR-07 | Epic 2 | Sending state / disabled input |
| FR-08 | Epic 2 | Inline 4xx/5xx error display |
| NFR-01 | Epic 2 | Vanilla HTML/JS constraint |
| NFR-04 | Epic 2 | Single-file constraint |
| FR-09 | Epic 3 | EventSource + KAYLEE_STREAM_ENABLED flag |

---

## Epic List

### Epic 1: Kaylee Test UI Foundation
Developer can reach the test UI in a browser, served directly by kaylee-agent.
**FRs covered:** FR-01, NFR-02, NFR-03

### Epic 2: Chat Session — Send, Receive, Display
Developer can start a conversation with Kaylee and see responses in a readable chat format.
**FRs covered:** FR-02, FR-03, FR-04, FR-05, FR-06, FR-07, FR-08, NFR-01, NFR-04

### Epic 3: Streaming-Ready Display Path
Developer can toggle real-time streaming display when kaylee-agent implements `POST /sessions/{id}/stream`.
**FRs covered:** FR-09

---

## Epic 1: Kaylee Test UI Foundation

Serve a static UI page from kaylee-agent's FastAPI app, gated by an environment variable so it is never exposed in production.

### Story 1.1: FastAPI StaticFiles mount with production env var gate

As a developer,
I want the chat UI served at `localhost:8000/ui` by kaylee-agent itself,
So that I can access the test interface without running any additional services, containers, or toolchain.

**Acceptance Criteria:**

**Given** `KAYLEE_DEV_UI_ENABLED=true` is set in the environment
**When** I navigate to `http://localhost:8000/ui`
**Then** the browser loads `ui/index.html` served by FastAPI

**Given** `KAYLEE_DEV_UI_ENABLED` is unset or set to `false`
**When** I navigate to `http://localhost:8000/ui`
**Then** the server returns 404 (route not mounted)

**Given** the StaticFiles mount is added
**When** I inspect `requirements.txt` and `pyproject.toml`
**Then** no new Python dependencies have been added (FastAPI StaticFiles is part of `fastapi[all]` or `starlette`)

**Given** the UI is served
**When** I open `ui/index.html` directly in the browser and via the `/ui` route
**Then** both render without errors in browser console

**Implementation notes:**
- Add to `app/main.py`: `if os.getenv("KAYLEE_DEV_UI_ENABLED", "false").lower() == "true": app.mount("/ui", StaticFiles(directory="ui", html=True), name="ui")`
- Create `ui/index.html` with minimal placeholder content (e.g. "Kaylee Test UI") — full UI content is Story 2.x
- Add `KAYLEE_DEV_UI_ENABLED=true` to `docker-compose.dev.yml` or local `.env`

---

### Story 1.2: Deploy chatui to kaylee-agent dev environment

As a developer,
I want the chatui served and reachable in the deployed dev environment,
So that I can use it to test Kaylee AI behaviour against live data without local port-forwarding.

**Acceptance Criteria:**

**Given** the Dockerfile is updated
**When** the image is built
**Then** `/app/ui/` is present in the container image (verified by `docker run --rm <image> ls /app/ui/`)

**Given** `terminus.infra` `values-dev.yaml` sets `devUI.enabled: true`
**When** ArgoCD syncs the dev deployment
**Then** the kaylee-agent pod has `KAYLEE_DEV_UI_ENABLED=true` in its environment

**Given** the dev deployment is synced
**When** I navigate to `https://fourdogs-kaylee.trantor.internal/ui`
**Then** the page returns HTTP 200 and renders the chat UI

**Given** the production values
**When** inspected
**Then** `devUI.enabled` is absent or `false` — the UI is not served in production

**Implementation notes:**
- Touches two repos: `fourdogs-kaylee-agent` (Dockerfile + Helm chart) AND `terminus.infra` (values-dev.yaml)
- Helm chart: add `devUI.enabled: false` to `values.yaml`; conditional `KAYLEE_DEV_UI_ENABLED` env var in `deployment.yaml`
- `terminus.infra`: `platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` — add `devUI.enabled: true`

---

## Epic 2: Chat Session — Send, Receive, Display

Full working chat UI: session creation, message send/receive, history display, error handling. Zero build toolchain. Single `index.html` file.

### Story 2.1: Session creation on page load and session ID display

As a developer,
I want a Kaylee session automatically created when I open the UI,
So that I can immediately start testing without any manual setup or configuration.

**Acceptance Criteria:**

**Given** I open `localhost:8000/ui`
**When** the page finishes loading
**Then** `POST /sessions` has been called and a `session_id` is returned

**Given** the session was created successfully
**When** I look at the page header
**Then** the session ID is visible (e.g. `Session: abc-123`)

**Given** `POST /sessions` returns an error (network failure or 5xx)
**When** the page loads
**Then** an inline error banner is displayed explaining that session creation failed, with the status code

**Given** I reload the page
**When** the page finishes loading
**Then** a new session ID is created (prior session is abandoned — expected behaviour)

**Implementation notes:**
- `ui/index.html` is the single file for all UI — no separate `app.js` or `style.css`
- Session ID held in a JS variable: `let sessionId = null`
- No `localStorage` or cookies — session is intentionally ephemeral

---

### Story 2.2: Send message and display sync response in chat format

As a developer,
I want to type a message, send it to Kaylee, and see the response displayed in a chat bubble format,
So that I can test Kaylee's AI behaviour and response quality directly.

**Acceptance Criteria:**

**Given** a session is active and I type a message in the input field
**When** I click Send or press Enter
**Then** `POST /sessions/{sessionId}/messages` is called with `{"message": "<my text>"}`

**Given** Kaylee returns a 200 response
**When** the response arrives
**Then** a user bubble with my message and an assistant bubble with Kaylee's response appear in the conversation area

**Given** the request is in flight
**When** I look at the input area
**Then** the input field and Send button are disabled and the status line reads "sending..."

**Given** the request completes (success or error)
**When** the response is processed
**Then** the input field and Send button are re-enabled and the status line returns to "ready"

**Given** the input field is empty
**When** I click Send
**Then** nothing is sent (no API call made)

**Given** `POST /sessions/{sessionId}/messages` returns a 4xx or 5xx
**When** the error is received
**Then** an inline error banner shows the status code and the user's message is NOT added to the conversation

**Implementation notes:**
- Response body: `{ "session_id": "...", "message_id": "...", "response": "..." }`
- Render `response` field as the assistant bubble text
- Vanilla HTML/JS — use `fetch()`, no XHR, no jQuery

---

### Story 2.3: Conversation history load and graceful empty state

As a developer,
I want to see any existing conversation history when I open the UI,
So that I can review the full session context and diagnose issues across page refreshes within the same session.

**Acceptance Criteria:**

**Given** I open the UI and a session is created
**When** `GET /sessions/{sessionId}/messages` is called on load
**Then** any prior messages in the session are rendered in the conversation area in chronological order

**Given** `GET /sessions/{sessionId}/messages` returns an empty array (new session)
**When** the response is processed
**Then** the conversation area is empty and no error is shown (graceful empty state)

**Given** `GET /sessions/{sessionId}/messages` returns a non-empty array
**When** the messages are rendered
**Then** user messages appear as user bubbles and assistant messages appear as assistant bubbles, matching the same formatting used for new messages

**Given** `GET /sessions/{sessionId}/messages` returns an error (4xx/5xx)
**When** the error is received
**Then** an error banner is displayed but the UI remains functional (user can still send new messages)

**Implementation notes:**
- History load fires after session creation succeeds
- Message role field: `"role": "user"` or `"role": "assistant"` — use to distinguish bubble type
- Call order: `POST /sessions` → success → `GET /sessions/{id}/messages` → render → enable input

---

## Epic 3: Streaming-Ready Display Path

Add the `EventSource` streaming path behind a feature flag, ready to activate when `POST /sessions/{id}/stream` is implemented in kaylee-agent.

### Story 3.1: EventSource streaming path behind KAYLEE_STREAM_ENABLED flag

As a developer,
I want to flip a single JS constant to switch the chat UI into streaming mode,
So that I can test Kaylee's real-time token streaming display as soon as the backend implements the stream endpoint.

**Acceptance Criteria:**

**Given** `KAYLEE_STREAM_ENABLED = false` (default)
**When** a message is sent
**Then** the sync path is used (`POST /sessions/{id}/messages`) — behaviour identical to Story 2.2

**Given** `KAYLEE_STREAM_ENABLED = true`
**When** a message is sent
**Then** `POST /sessions/{sessionId}/stream` is called with `{"message": "<text>"}` and an `EventSource` connection is opened

**Given** the EventSource connection is open and tokens arrive
**When** each `data:` event is received
**Then** the token is appended to the assistant bubble progressively (text appears as it streams)

**Given** the stream sends a `done` event
**When** the event is received
**Then** the EventSource is closed, the assistant bubble is complete, and the input is re-enabled

**Given** the stream sends an `error` event or the connection drops
**When** the error is received
**Then** the EventSource is closed, an inline error banner is displayed, and the input is re-enabled

**Given** `KAYLEE_STREAM_ENABLED = true` but the server returns 501
**When** the response arrives
**Then** an inline error banner shows "Streaming not yet implemented (501)" and input is re-enabled

**Implementation notes:**
- `const KAYLEE_STREAM_ENABLED = false;` at the top of the script block — single line to toggle
- EventSource on POST is non-standard — use `fetch()` with streaming response body (`response.body.getReader()`) or a library-free SSE approach, not native `EventSource` (which is GET-only)
- Alternatively, if kaylee-agent implements SSE as a GET endpoint with session/message params, native `EventSource` applies — match the endpoint spec when it lands
- This story is considered done when the flag path is wired and tested with a mock or the real endpoint when available
