# Story 5.3: /readyz wires HealthCheck

Status: ready-for-dev

## Story

As an **operator**,
I want the gateway's `/readyz` endpoint to include routing layer health in its response,
So that orchestration infrastructure (k3s, ArgoCD) can detect when all providers are exhausted.

## Acceptance Criteria

1. **Given** the `/readyz` handler exists in the gateway  
   **When** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: false`  
   **Then** `/readyz` returns HTTP 200 and the routing section of the response shows healthy

2. **Given** `router.HealthCheck(ctx)` returns `HealthStatus.AllExhausted: true`  
   **When** `/readyz` is called  
   **Then** it returns HTTP 503 — the gateway reports itself unhealthy  
   **And** the response body includes per-provider budget state from `HealthStatus.Providers`

## Tasks / Subtasks

- [ ] Locate the existing `/readyz` handler in the gateway (AC: 1, 2)
- [ ] Call `h.router.HealthCheck(ctx)` in the `/readyz` handler (AC: 1, 2)
- [ ] Add routing health to `/readyz` response (AC: 1, 2)
  - [ ] Include `routing: { providers: [...], all_exhausted: bool, degraded: bool }` in response JSON
  - [ ] `ProviderStatus` fields: `name`, `budget_limit_usd`, `budget_used_usd`, `budget_exhausted`
- [ ] Conditional HTTP status (AC: 2)
  - [ ] If `healthStatus.AllExhausted == true`: return HTTP 503
  - [ ] Otherwise: return HTTP 200 (or whatever the existing health success code is)
- [ ] Verify `go build ./...` passes

## Dev Notes

- The `/readyz` handler already exists in the gateway — this story extends it, not replaces it
- HTTP status logic: if ANY component is unhealthy → 503; if routing layer is the only failing component, that alone should trigger 503 (FR21)
- Response body must NOT contain API keys or credential values (NFR-S1) — `HealthStatus.Providers` has only budget metrics, which is safe
- `HealthCheck` completes in < 5ms (NFR-P3) — safe for synchronous use in `/readyz` handler
- If the gateway has other health checks (database, external services), combine the `AllExhausted` check with existing unhealthy conditions using OR logic

### Project Structure Notes

- File: `/readyz` handler in `internal/handlers/` or `cmd/gateway/` (confirm from codebase)
- The `router` field on the server/handler struct was added in Story 5.1

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR20, FR21] — HealthCheck shape; /readyz → 503 when all exhausted
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — HealthCheck wiring
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 5.3] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
