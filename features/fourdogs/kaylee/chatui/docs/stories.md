# Stories — fourdogs-kaylee-chatui

Cross-reference of all stories across epics. Full story text and acceptance criteria are in `epics.md`.
Dev-ready story files are in the `stories/` subdirectory.

## Sprint 1

| Story | Title | Status |
|-------|-------|--------|
| 1.1 | FastAPI StaticFiles mount with production env var gate | ready-for-dev |
| 1.2 | Deploy chatui to kaylee-agent dev environment | ready-for-dev |
| 2.1 | Session creation on page load and session ID display | ready-for-dev |
| 2.2 | Send message and display sync response in chat format | ready-for-dev |
| 2.3 | Conversation history load and graceful empty state | ready-for-dev |
| 3.1 | EventSource streaming path behind KAYLEE_STREAM_ENABLED flag | ready-for-dev |

## Story Order

Recommended execution order (dependencies):
1. Story 1.1 — Foundation (required for all other stories)
2. Story 1.2 — Deploy to dev env (requires 1.1; touches kaylee-agent + terminus.infra)
3. Story 2.1 — Session lifecycle (required for 2.2 and 2.3)
4. Story 2.2 — Core send/receive (independent after 2.1)
5. Story 2.3 — History display (independent after 2.1, can parallel 2.2)
6. Story 3.1 — Streaming path (independent, deferrable until backend ready)
