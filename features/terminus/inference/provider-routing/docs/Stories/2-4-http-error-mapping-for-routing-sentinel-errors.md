# Story 2.4: HTTP error mapping for routing sentinel errors

Status: ready-for-dev

## Story

As a **gateway developer**,
I want `internal/errors` to map routing sentinel errors to the correct HTTP status codes,
So that callers receive 503 for no-eligible-provider and 400 for invalid-workload-class without custom handler logic.

## Acceptance Criteria

1. **Given** an error where `errors.Is(err, routing.ErrNoEligibleProvider)` is true  
   **When** it reaches the gateway's error mapping layer in `internal/errors`  
   **Then** it maps to `HTTP 503 Service Unavailable`

2. **Given** an error where `errors.Is(err, routing.ErrInvalidWorkloadClass)` is true  
   **When** it reaches the gateway's error mapping layer  
   **Then** it maps to `HTTP 400 Bad Request`

3. **Given** a wrapped error `fmt.Errorf("routing: no eligible provider for profile %q: %w", profile, ErrNoEligibleProvider)`  
   **When** `errors.Is(err, routing.ErrNoEligibleProvider)` is evaluated  
   **Then** it returns `true` (wrapping must not break sentinel matching)

## Tasks / Subtasks

- [ ] Locate the existing error mapping layer in `internal/errors` of the gateway (AC: 1, 2)
- [ ] Add routing sentinel error cases to the HTTP status mapping (AC: 1, 2)
  - [ ] `routing.ErrNoEligibleProvider` → `http.StatusServiceUnavailable` (503)
  - [ ] `routing.ErrInvalidWorkloadClass` → `http.StatusBadRequest` (400)
- [ ] Verify wrapping does not break `errors.Is` (AC: 3)
  - [ ] Use `errors.Is(wrappedErr, routing.ErrNoEligibleProvider)` in a test — must return `true`
- [ ] Confirm import path: `internal/routing` imported by `internal/errors` must not create a circular import

## Dev Notes

- This story touches `internal/errors` package in the gateway — NOT `internal/routing/`
- The gateway's error mapping layer likely uses a switch on error type or a map/slice of `(predicate, statusCode)` pairs; add the routing sentinels using the same pattern already in use
- Import cycle risk: `internal/errors` imports `internal/routing` — verify the existing dep graph supports this. If `internal/errors` is already imported by `internal/routing`, this creates a cycle; resolve by moving sentinel definitions to a separate `internal/routing/errors` sub-package or by checking the existing mapping approach
- Error wrapping with `%w` preserves the sentinel chain — `errors.Is` unwraps automatically through any depth of `fmt.Errorf("%w")` wrapping (AC: 3)
- Sentinel variables in `errors.go` are pointer-comparable `error` values (`var ErrX = errors.New("...")`) — same pointer throughout program lifetime; `errors.Is` uses `==` at the base level after unwrapping

### Project Structure Notes

- Primary file: `internal/errors/` package in the gateway (NOT `internal/routing/`)
- Check gateway's `internal/errors` for existing convention (likely `HTTPStatusCode(err error) int` or a middleware handler)
- Import: `"github.com/..../terminus-inference-gateway/internal/routing"` (confirm the module path from `go.mod`)

### References

- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — "HTTP error mapping additions to `internal/errors`: `ErrNoEligibleProvider→503`, `ErrInvalidWorkloadClass→400`"
- [Source: docs/terminus/inference/provider-routing/prd.md#FR7] — `ErrNoEligibleProvider` on no eligible provider
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.4] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
