# Story 2.3: Resolve returns clean sentinel errors for invalid and exhausted states

**Initiative:** terminus-inference-provider-routing
**Epic:** 2 — Gateway can route requests to the correct provider
**Status:** review

## Story

As a **gateway developer**,
I want `Resolve` to return typed sentinel errors for unrecognized workload classes and when no eligible provider exists,
So that callers can map these to correct HTTP status codes without inspecting error strings.

## Acceptance Criteria

**Given** a `RoutingRequest` with `Profile: "unknown-class"` not defined in any loaded profile
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrInvalidWorkloadClass)` is true
**And** the error message includes the unrecognized profile name

**Given** all providers in the chain for the requested profile are unavailable
**When** `Resolve` is called
**Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true
**And** no panic occurs under any input

**Given** a panic is induced inside `Resolve` (e.g. via test injection)
**When** the panic fires
**Then** the deferred recover catches it, logs an `slog.Error` with the recovered value, and returns `(nil, ErrNoEligibleProvider)`
**And** the gateway process does not crash

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Implement in `router.go` (additions to `Resolve`)
- Panic recovery pattern:
  ```go
  func (r *router) Resolve(ctx context.Context, req *RoutingRequest) (resp *RoutingResponse, err error) {
      defer func() {
          if rec := recover(); rec != nil {
              slog.ErrorContext(ctx, "routing: resolve panic recovered", "recovered", rec)
              err = ErrNoEligibleProvider
          }
      }()
      // ... resolve logic
  }
  ```
- Named return `err` required for panic recovery to work
- Error wrapping: `fmt.Errorf("routing: resolve: invalid workload class %q: %w", req.Profile, ErrInvalidWorkloadClass)`

## Dependencies

- Story 2.2 (Resolve base implementation)
