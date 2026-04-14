# Story 5.2: HTTP handler calls Resolve and RecordUsage

**Initiative:** terminus-inference-provider-routing
**Epic:** 5 — Gateway can deploy with routing enabled
**Status:** review

## Story

As a **gateway developer**,
I want the primary inference HTTP handler to call `Resolve` before dispatching and `RecordUsage` after receiving the provider response,
So that every inference request flows through the routing layer.

## Acceptance Criteria

**Given** an inbound inference request with `X-Workload-Class: interactive` (or equivalent routing signal)
**When** the handler processes the request
**Then** `router.Resolve(ctx, &RoutingRequest{Profile: workloadClass})` is called before provider dispatch
**And** if `Resolve` returns `ErrNoEligibleProvider`, the handler returns HTTP 503
**And** if `Resolve` returns `ErrInvalidWorkloadClass`, the handler returns HTTP 400

**Given** the provider returns a successful response with `usage.PromptTokens` and `usage.CompletionTokens`
**When** the handler finishes reading the response
**Then** `router.RecordUsage(ctx, providerName, usage.PromptTokens, usage.CompletionTokens)` is called
**And** `RecordUsage` is called on every successful response — not only when degraded

**Given** the provider response includes degradation info (`RoutingResponse.Degraded == true`)
**When** the handler writes its response to the caller
**Then** the response includes `degraded: true`, `fallback_provider`, and `degradation_reason` fields

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Modify the existing inference handler (likely in `internal/handler/` or similar)
- Workload class source: `X-Workload-Class` request header; default to `"batch"` if not present
- On `Resolve` error: use `internal/errors` HTTP mapping (added in story 2.4) — no handler-specific status codes
- `RecordUsage` nil-usage: if Provider response has no usage object, pass `(ctx, providerName, 0, 0)` — budget.go handles the len/4 estimate (story 3.2)
- Degradation fields in response: add to existing OpenAI-compatible response envelope under a `_routing` key or equivalent extension field

## Dependencies

- Story 5.1 (Router injected into server)
- Stories 2.4, 3.2, 3.3 (full routing pipeline)
