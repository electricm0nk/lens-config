---
initiative: chatui
track: tech-change
phase: devproposal
created: 2026-04-13
note: "Generated retroactively — tech-change track skipped businessplan phase. Captures known context before devproposal."
---

# Product Brief — fourdogs-kaylee-chatui

## Problem

Developers iterating on the kaylee-agent AI service (session management, prompt quality, response behaviour) currently have no way to test it directly. The only client is the fourdogs-central app, which adds a full deployment dependency and hides the raw Kaylee response behind an SSE proxy wrapper. This slows feedback loops during development.

## Solution

A minimal developer chat UI served by kaylee-agent itself — no external deployment required. Open `localhost:8000/ui`, create a session, start chatting. Direct REST calls on the same origin, zero config.

## Goals

- Enable direct conversation testing with Kaylee during development
- Expose raw session/message API responses without abstraction layers
- Future-proof for streaming when `POST /sessions/{id}/stream` is implemented

## Non-Goals

- Not a user-facing product
- No auth, no multi-user support, no mobile layout
- No production deployment (gated behind env var)

## Success Criteria

- Developer can open `localhost:8000/ui` and send a message to Kaylee
- Response is displayed in a readable chat format
- Session persists across messages in the same tab session
- No additional containers, ports, or toolchain required

## Scope

Single HTML file + FastAPI `StaticFiles` mount. Estimated: 1 epic, 3 stories.
