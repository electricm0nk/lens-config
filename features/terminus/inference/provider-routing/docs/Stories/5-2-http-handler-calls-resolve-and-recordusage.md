# Story 5.2: HTTP handler calls Resolve and RecordUsage

Status: ready-for-dev

## Story

As a **gateway developer**,
I want the primary inference HTTP handler to call `Resolve` before dispatching and `RecordUsage` after receiving the provider response,
So that every inference request flows through the routing layer.

## Acceptance Criteria

1. **Given** an inbound inference request with `X-Workload-Class: interactive` (or equivalent routing signal)  
   **When** the handler processes the request  
   **Then** `router.Resolve(ctx, &RoutingRequest{Profile: workloadClass})` is called before provider dispatch  
   **And** if `Resolve` returns `ErrNoEligibleProvider`, the handler returns HTTP 503  
   **And** if `Resolve` returns `ErrInvalidWorkloadClass`, the handler returns HTTP 400

2. **Given** the provider returns a successful response with `usage.PromptTokens` and `usage.CompletionTokens`  
   **When** the handler finishes reading the response  
   **Then** `router.RecordUsage(ctx, providerName, usage.PromptTokens, usage.CompletionTokens)` is called  
   **And** `RecordUsage` is called on every successful response — not only when degraded

3. **Given** the provider response includes degradation info (`RoutingResponse.Degraded == true`)  
   **When** the handler writes its response to the caller  
   **Then** the response includes `degraded: true`, `fallback_provider`, and `degradation_reason` fields

## Tasks / Subtasks

- [ ] Locate the primary inference HTTP handler in the gateway (AC: 1)
- [ ] Add `Resolve` call before provider dispatch (AC: 1)
  - [ ] Extract workload class from `X-Workload-Class` header (or existing routing signal in the gateway)
  - [ ] `routingResp, err := h.router.Resolve(ctx, &routing.RoutingRequest{Profile: workloadClass})`
  - [ ] Handle errors via `internal/errors` HTTP mapping (AC: 1)
- [ ] Select provider by `routingResp.Provider` name for dispatch (AC: 1)
- [ ] After provider response: call `RecordUsage` with actual usage (AC: 2)
  - [ ] If usage is nil: set body length in context (`context.WithValue(ctx, bodyLenContextKey, len(rawBody))`) before calling `RecordUsage` with 0,0 — triggers fallback estimate in RecordUsage (Story 3.2)
- [ ] Include degradation fields in outbound response when `routingResp.Degraded == true` (AC: 3)
- [ ] Verify `go build ./...` passes

## Dev Notes

- `X-Workload-Class` header: confirm the actual header name used in the gateway — may differ; check existing handler code
- If no workload class header is present: default to `"batch"` or return 400 — confirm desired behavior with gateway conventions
- Provider selection: after `Resolve`, use `routingResp.Provider` to look up the correct `Provider` from the gateway's provider registry (not from `internal/routing`)
- The `bodyLenContextKey` is the unexported context key type defined in `internal/routing` in Story 3.2 — it cannot be used directly from the handler package; consider exporting a `WithBodyLen(ctx, n int) context.Context` helper from `internal/routing` for handlers to call
- `RecordUsage` is non-blocking and returns no error — always call it, do not select on a channel or wrap in goroutine
- Degradation fields in outbound response: follow the existing gateway response format (likely JSON); add `routing_degraded`, `routing_fallback_provider`, `routing_degradation_reason` fields or match existing conventions

### Project Structure Notes

- File: primary inference handler in `internal/handlers/` or similar (confirm path from gateway codebase)
- Import: `"github.com/..../terminus-inference-gateway/internal/routing"` for types

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR1, FR5, FR6] — Gateway calls Resolve; degradation fields in response
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — HTTP handler integration
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 5.2] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
