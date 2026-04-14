# Story 2.1: RoutingRequest, RoutingResponse, and shared types

**Initiative:** terminus-inference-provider-routing
**Epic:** 2 — Gateway can route requests to the correct provider
**Status:** review

## Story

As a **gateway developer**,
I want the shared request/response types for the routing package defined in `types.go`,
So that `Resolve` and all callers have a single canonical type definition to import.

## Acceptance Criteria

**Given** `types.go` is implemented
**Then** `RoutingRequest` contains at minimum: `Profile string` (workload class name)
**And** `RoutingResponse` contains: `Provider string`, `Degraded bool`, `FallbackProvider string`, `DegradationReason string`
**And** `HealthStatus` contains: `Providers []ProviderStatus`, `Degraded bool`, `AllExhausted bool`
**And** `ProviderStatus` contains: `Name string`, `BudgetLimitUSD float64`, `BudgetUsedUSD float64`, `BudgetExhausted bool`
**And** `DegradationReasonBudgetExhausted` and `DegradationReasonUnhealthy` string constants are declared
**And** all types compile cleanly with no circular imports

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `internal/routing/types.go`
- `RoutingRequest.Profile` is the workload class name (maps to a `profiles[].name` entry from config)
- `RoutingResponse.FallbackProvider` — only populated when `Degraded: true`
- `RoutingResponse.DegradationReason` — use `DegradationReasonBudgetExhausted` or `DegradationReasonUnhealthy` constants
- `HealthStatus.AllExhausted` — true when ALL providers across ALL profiles are exhausted
- `ProviderStatus.BudgetLimitUSD: 0` means unlimited (not zero-dollar limit)
- No methods on types — plain structs only

## Dependencies

- Story 1.1 (package must compile)
