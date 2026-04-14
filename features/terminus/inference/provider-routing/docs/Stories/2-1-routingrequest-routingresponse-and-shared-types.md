# Story 2.1: RoutingRequest, RoutingResponse, and shared types

Status: ready-for-dev

## Story

As a **gateway developer**,
I want the shared request/response types for the routing package defined in `types.go`,
So that `Resolve` and all callers have a single canonical type definition to import.

## Acceptance Criteria

1. **Given** `types.go` is implemented  
   **Then** `RoutingRequest` contains at minimum: `Profile string` (workload class name)

2. **And** `RoutingResponse` contains: `Provider string`, `Degraded bool`, `FallbackProvider string`, `DegradationReason string`

3. **And** `HealthStatus` contains: `Providers []ProviderStatus`, `Degraded bool`, `AllExhausted bool`

4. **And** `ProviderStatus` contains: `Name string`, `BudgetLimitUSD float64`, `BudgetUsedUSD float64`, `BudgetExhausted bool`

5. **And** `DegradationReasonBudgetExhausted` and `DegradationReasonUnhealthy` string constants are declared

6. **And** all types compile cleanly with no circular imports

## Tasks / Subtasks

- [ ] Implement `types.go` in `internal/routing/` (AC: 1–6)
  - [ ] `type RoutingRequest struct { Profile string }` (AC: 1)
  - [ ] `type RoutingResponse struct { Provider string; Degraded bool; FallbackProvider string; DegradationReason string }` (AC: 2)
  - [ ] `type HealthStatus struct { Providers []ProviderStatus; Degraded bool; AllExhausted bool }` (AC: 3)
  - [ ] `type ProviderStatus struct { Name string; BudgetLimitUSD float64; BudgetUsedUSD float64; BudgetExhausted bool }` (AC: 4)
  - [ ] `const DegradationReasonBudgetExhausted = "budget_exhausted"` (AC: 5)
  - [ ] `const DegradationReasonUnhealthy = "provider_unhealthy"` (AC: 5)
- [ ] Update `router.go` `Router` interface stubs to reference these concrete types (replacing any placeholder stubs from Story 1.1) (AC: 6)
- [ ] Verify `go build ./internal/routing/...` still passes (AC: 6)

## Dev Notes

- These are the exported types in the `internal/routing/` package — they form the public API of the routing layer
- `RoutingRequest.Profile` maps to the workload class name from gateway context (e.g., `"interactive"`, `"batch"`) — same as the key in the `profiles` YAML section
- `DegradationReasonBudgetExhausted = "budget_exhausted"` — exact string used in story 3.3 when fallback occurs
- `DegradationReasonUnhealthy = "provider_unhealthy"` — reserved for future use (not activated in MVP stories); define it now per architecture spec
- `HealthStatus.AllExhausted` — set to `true` when ALL providers are budget-exhausted, triggering `/readyz` → 503 (Story 5.3)
- No methods on any of these structs — pure data types; behavior lives in the `Router` implementation
- If Story 1.1 created placeholder empty structs for these types, replace them completely in this story

### Project Structure Notes

- File: `internal/routing/types.go`
- All types defined here are in `package routing` — no sub-packages
- `ProviderStatus` fields use `USD` suffix to clarify currency; `BudgetUsedUSD` tracks accumulated dollar-equivalent spend

### References

- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — Exact struct shapes for `RoutingRequest`, `RoutingResponse`, `HealthStatus`, `ProviderStatus`; degradation constants
- [Source: docs/terminus/inference/provider-routing/prd.md#FR6, FR20] — Degradation fields in response; HealthCheck struct shape
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.1] — All AC fields

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
