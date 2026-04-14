# Story 2.2: Send message and display sync response in chat format

Status: ready-for-dev

## Story

As a developer,
I want to type a message, send it to Kaylee, and see the response displayed in a chat bubble format,
so that I can test Kaylee's AI behaviour and response quality directly.

## Acceptance Criteria

1. Typing in the input field and clicking Send (or pressing Enter) calls `POST /sessions/{sessionId}/messages` with `{"message": "<text>"}`
2. On 200: both a user bubble and an assistant bubble (with Kaylee's response) appear in the conversation area
3. While request is in flight: input and Send button are disabled; status line reads "sending..."
4. After response (success or error): input and Send button are re-enabled; status line returns to "ready"
5. Empty input: no API call is made
6. On 4xx/5xx: an inline error banner shows the status code; user's message is NOT added to the conversation

## Tasks / Subtasks

- [ ] Implement `sendMessage()` async function (AC: 1, 2, 3, 4, 5, 6)
  - [ ] Guard: if `input.value.trim() === ""` return early (AC: 5)
  - [ ] Disable input + button, set status "sending..." (AC: 3)
  - [ ] `POST /sessions/${sessionId}/messages` with `{"message": trimmedText}`
  - [ ] On 200: call `appendMessage("user", text)` and `appendMessage("assistant", response.response)` (AC: 2)
  - [ ] On non-200: call `showError("Error " + status)` — do NOT append user message (AC: 6)
  - [ ] Finally: re-enable input + button, set status "ready" (AC: 4)
- [ ] Implement `appendMessage(role, text)` function (AC: 2)
  - [ ] Creates a div with class `message-${role}` (user or assistant)
  - [ ] Appends to conversation area
  - [ ] Scrolls conversation area to bottom
- [ ] Wire Send button click and input `keydown` Enter to `sendMessage()` (AC: 1)
- [ ] Implement `showError(message)` function (AC: 6)
  - [ ] Shows error banner, auto-hides after 5 seconds or on next send
- [ ] Implement `setStatus(text)` and `setInputEnabled(bool)` helpers (AC: 3, 4)
- [ ] Add minimal CSS for user/assistant bubbles (inline `<style>` block) (NFR-01, NFR-04)
  - [ ] User bubble: right-aligned, light background
  - [ ] Assistant bubble: left-aligned, distinct background
  - [ ] Status line: small muted text below input

## Dev Notes

- **POST /sessions/{id}/messages response shape:** `{ "session_id": "...", "message_id": "...", "response": "..." }` — use `.response` field for assistant bubble text
- **Input guard:** check BOTH `input.value.trim() === ""` AND `sessionId === null` before sending
- **Disable/enable pattern:** `input.disabled = true; sendBtn.disabled = true` before fetch, always re-enable in `finally` block
- **Status flow:** "ready" → "sending..." → "ready" (on success or error)
- **Error banner:** reuse the banner introduced in Story 2.1 — same `showError()` function
- **Scroll to bottom:** `conversationDiv.scrollTop = conversationDiv.scrollHeight` after append
- **Plain text rendering:** render message text as `textContent` not `innerHTML` — avoids XSS on AI output
- **No streaming in this story** — `KAYLEE_STREAM_ENABLED` constant (introduced in Story 3.1) is irrelevant here; this story is sync-only

### Project Structure Notes

- All changes to `ui/index.html` — no new files
- CSS goes in the existing/new inline `<style>` block
- JS goes in the existing inline `<script>` block

### References

- Architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — Send Message (Sync) flow, D1, D3
- Tech Decisions: `docs/fourdogs/kaylee/chatui/tech-decisions.md` — TDL-001, TDL-003, TDL-004
- PRD: `docs/fourdogs/kaylee/chatui/prd.md` — FR-04, FR-05, FR-07, FR-08, NFR-01, NFR-04

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `ui/index.html` (modified — sendMessage, appendMessage, showError, setStatus, CSS bubbles)
