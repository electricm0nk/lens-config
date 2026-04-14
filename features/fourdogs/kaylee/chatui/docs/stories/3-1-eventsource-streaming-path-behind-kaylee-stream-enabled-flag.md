# Story 3.1: EventSource streaming path behind KAYLEE_STREAM_ENABLED flag

Status: ready-for-dev

## Story

As a developer,
I want to flip a single JS constant to switch the chat UI into streaming mode,
so that I can test Kaylee's real-time token streaming display as soon as the backend implements the stream endpoint.

## Acceptance Criteria

1. A `const KAYLEE_STREAM_ENABLED = false` constant exists at the top of the script block
2. When `KAYLEE_STREAM_ENABLED = false`: behavior is identical to Story 2.2 — sync path, no change
3. When `KAYLEE_STREAM_ENABLED = true`: `POST /sessions/{sessionId}/stream` is called instead of `/messages`
4. With streaming enabled: tokens append progressively to the assistant bubble as server-sent events arrive
5. When the server sends a `done` event (or closes the stream): the EventSource closes and input is re-enabled
6. On stream error or connection drop: error banner displayed, EventSource closed, input re-enabled
7. If `/stream` returns 501: error banner reads "Streaming not yet implemented (501)"

## Tasks / Subtasks

- [ ] Add `const KAYLEE_STREAM_ENABLED = false;` at top of script block (AC: 1)
- [ ] Refactor `sendMessage()` to branch on `KAYLEE_STREAM_ENABLED` (AC: 2, 3)
  - [ ] `if (!KAYLEE_STREAM_ENABLED) { return sendMessageSync(text); }`
  - [ ] `else { return sendMessageStream(text); }`
  - [ ] Extract existing sync logic from Story 2.2 into `sendMessageSync(text)`
- [ ] Implement `sendMessageStream(text)` (AC: 3, 4, 5, 6, 7)
  - [ ] Disable input + button, set status "streaming..."
  - [ ] Create assistant bubble immediately (empty, will fill progressively)
  - [ ] Use `fetch()` with streaming body reader: `const reader = response.body.getReader()`
  - [ ] Decode chunks: `new TextDecoder().decode(value)` and append to assistant bubble `textContent`
  - [ ] Parse SSE lines: lines starting with `data:` → extract payload, append text to bubble
  - [ ] Detect `data: [DONE]` or `event: done` → close reader, re-enable input, set status "ready"
  - [ ] On `response.status === 501`: show "Streaming not yet implemented (501)", re-enable input (AC: 7)
  - [ ] On fetch error or read error: `showError(...)`, re-enable input (AC: 6)
- [ ] Test with `KAYLEE_STREAM_ENABLED = false` — verify no regression to Story 2.2 behavior (AC: 2)

## Dev Notes

- **Why `fetch()` not native `EventSource`:** Native `EventSource` is GET-only. `POST /sessions/{id}/stream` is a POST. Use `fetch()` with `response.body.getReader()` for SSE-over-POST. [Source: architecture.md — D4, Story 3.1 implementation note]
- **SSE wire format over fetch:**
  ```
  data: token text here\n\n
  data: more tokens\n\n
  data: [DONE]\n\n
  ```
  Split on `\n\n`, filter lines starting with `data:`, strip `data: ` prefix, append.
- **If kaylee-agent uses GET for streaming:** When the endpoint is actually implemented, confirm whether it's GET or POST. If GET, native `EventSource` applies. Check the endpoint spec before implementing — the `KAYLEE_STREAM_ENABLED` flag defers this decision.
- **Progressive append:** append to `assistantBubble.textContent += chunk` not `innerHTML` (XSS safety)
- **Error recovery:** all error paths MUST re-enable input — use try/finally pattern
- **Refactor only `sendMessage()`:** no other functions from Stories 2.1-2.3 should need changes
- **Open item OI-01:** Verify `POST /sessions/{id}/stream` response format before implementing — stream event structure not yet confirmed. If backend isn't ready, this story's streaming path can be tested with a mock endpoint.

### Project Structure Notes

- All changes to `ui/index.html` — `KAYLEE_STREAM_ENABLED` constant + `sendMessageStream()` function + `sendMessage()` refactor
- Zero new files

### References

- Architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — D4, Send Message (Stream — future) flow
- Tech Decisions: `docs/fourdogs/kaylee/chatui/tech-decisions.md` — TDL-004
- PRD: `docs/fourdogs/kaylee/chatui/prd.md` — FR-09
- Readiness checklist: `docs/fourdogs/kaylee/chatui/readiness-checklist.md` — OI-01 (verify stream endpoint format)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `ui/index.html` (modified — KAYLEE_STREAM_ENABLED const, sendMessageStream, sendMessage refactor)
