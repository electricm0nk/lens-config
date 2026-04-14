# Story 2.3: Conversation history load and graceful empty state

Status: ready-for-dev

## Story

As a developer,
I want to see any existing conversation history when I open the UI,
so that I can review the full session context and diagnose issues across page refreshes within the same session.

## Acceptance Criteria

1. After session creation, `GET /sessions/{sessionId}/messages` is called
2. If the response is an empty array: the conversation area is empty with no error
3. If the response contains messages: they are rendered in chronological order matching the same bubble format as Story 2.2
4. User messages (role: `"user"`) render as user bubbles; assistant messages (role: `"assistant"`) render as assistant bubbles
5. If `GET /sessions/{id}/messages` returns an error (4xx/5xx): an error banner is shown but the UI remains functional

## Tasks / Subtasks

- [ ] Implement `renderMessages(messages)` function (AC: 2, 3, 4)
  - [ ] If `messages.length === 0`: return early (no output, no error — AC: 2)
  - [ ] For each message in array: call `appendMessage(message.role, message.content)` (AC: 3, 4)
  - [ ] Relies on `appendMessage()` from Story 2.2 — no duplication
- [ ] Wire `loadHistory()` (stubbed in Story 2.1) to call `renderMessages()` on success (AC: 1, 2, 3)
  - [ ] `GET /sessions/${sessionId}/messages`
  - [ ] On 200: `renderMessages(data)` where `data` is the response array
  - [ ] On 4xx/5xx: `showError("Could not load history: " + status)` — UI stays functional (AC: 5)
- [ ] Verify call order: `createSession` → success → `loadHistory` → `renderMessages` (AC: 1)

## Dev Notes

- **GET /sessions/{id}/messages response shape:** `[{ "role": "user"|"assistant", "content": "..." }]` — top-level array, not an object
- **Empty state is NOT an error:** `[]` response is normal for a brand-new session (adversarial review gate note)
- **`appendMessage()` reuse:** do NOT reimplement bubble rendering — call the same function from Story 2.2. This story is purely wiring `loadHistory` to `renderMessages`
- **Scroll after render:** after `renderMessages()` completes, scroll conversation area to bottom
- **Error handling:** use the existing `showError()` function from Story 2.2 — same banner, same pattern
- **Input stays enabled:** a history load failure does not block the user from sending messages

### Project Structure Notes

- All changes to `ui/index.html` — `renderMessages()` function and `loadHistory()` completion
- Minimal change: `renderMessages` is 5-8 lines; `loadHistory` fetch is already stubbed from Story 2.1

### References

- Architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — Session Initialization (GET history step)
- PRD: `docs/fourdogs/kaylee/chatui/prd.md` — FR-06, FR-08
- Adversarial review: `docs/fourdogs/kaylee/chatui/adversarial-review-report.md` — informational note on empty-state

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `ui/index.html` (modified — renderMessages function, loadHistory fetch completion)
