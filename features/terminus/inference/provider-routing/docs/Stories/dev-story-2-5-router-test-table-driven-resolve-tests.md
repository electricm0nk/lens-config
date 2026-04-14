# Story 2.5: router_test.go — table-driven Resolve tests

**Initiative:** terminus-inference-provider-routing
**Epic:** 2 — Gateway can route requests to the correct provider
**Status:** review

## Story

As a **developer**,
I want `router_test.go` to cover all `Resolve` paths with table-driven tests and the race detector,
So that routing logic is regression-safe and concurrent access is validated.

## Acceptance Criteria

**Given** `router_test.go` uses table-driven tests in `package routing` (white-box)
**When** `go test -race ./internal/routing/...` is run
**Then** all cases pass: happy path (first provider), fallback (second provider), invalid profile, all-unavailable, panic recovery
**And** mock providers implement the gateway `Provider` interface — no test-only interfaces are created
**And** all test cases use deterministic, non-network inputs

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `internal/routing/router_test.go`
- `package routing` — white-box; can access unexported fields directly
- Test cases:
  1. Happy path — first provider eligible → returns first provider, Degraded: false
  2. Fallback — first provider exhausted → returns second provider, Degraded: false (exhaustion degradation handled in story 3.3)
  3. Invalid profile — returns ErrInvalidWorkloadClass
  4. All unavailable — returns ErrNoEligibleProvider
  5. Panic recovery — inject panic via test helper, verify ErrNoEligibleProvider returned, process not crashed
  6. Concurrent resolve — 100 goroutines, same profile, no race detection
- Use `testing.T.Parallel()` on concurrent test
- No network calls — all provider state is in-memory

## Dependencies

- Stories 2.2 and 2.3 (Resolve fully implemented)
