# Story 2.3: Resolve returns clean sentinel errors for invalid and exhausted states

Status: ready-for-dev

## Story

As a **gateway developer**,
I want `Resolve` to return typed sentinel errors for unrecognized workload classes and when no eligible provider exists,
So that callers can map these to correct HTTP status codes without inspecting error strings.

## Acceptance Criteria

1. **Given** a `RoutingRequest` with `Profile: "unknown-class"` not defined in any loaded profile  
   **When** `Resolve` is called  
   **Then** it returns `(nil, err)` where `errors.Is(err, ErrInvalidWorkloadClass)` is true  
   **And** the error message includes the unrecognized profile name

2. **Given** all providers in the chain for the requested profile are unavailable  
   **When** `Resolve` is called  
   **Then** it returns `(nil, err)` where `errors.Is(err, ErrNoEligibleProvider)` is true  
   **And** no panic occurs under any input

3. **Given** a panic is induced inside `Resolve` (e.g. via test injection)  
   **When** the panic fires  
   **Then** the deferred recover catches it, logs an `slog.Error` with the recovered value, and returns `(nil, ErrNoEligibleProvider)`  
   **And** the gateway process does not crash

## Tasks / Subtasks

- [ ] Add unknown-profile error path to `Resolve` (AC: 1)
  - [ ] If `req.Profile` not in `profiles` map: `return nil, fmt.Errorf("routing: resolve: unknown workload class %q: %w", req.Profile, ErrInvalidWorkloadClass)`
- [ ] Add all-unavailable error path to `Resolve` (AC: 2)
  - [ ] After iterating all providers with no eligible one found: `return nil, fmt.Errorf("routing: resolve: no eligible provider for profile %q: %w", req.Profile, ErrNoEligibleProvider)`
- [ ] Add panic recovery via named-return defer/recover to `Resolve` (AC: 3)
  - [ ] Use named return: `func (r *router) Resolve(ctx context.Context, req *RoutingRequest) (resp *RoutingResponse, err error)`
  - [ ] `defer func() { if rec := recover(); rec != nil { slog.ErrorContext(ctx, "routing: resolve: panic recovered", "recovered", fmt.Sprintf("%v", rec)); err = ErrNoEligibleProvider } }()`
- [ ] Verify: `errors.Is(wrappedErr, ErrInvalidWorkloadClass)` returns true (AC: 1)
- [ ] Verify: `errors.Is(wrappedErr, ErrNoEligibleProvider)` returns true (AC: 2)
- [ ] Verify: panic injection test passes with no crash (AC: 3)

## Dev Notes

- Error wrapping pattern: `fmt.Errorf("routing: resolve: <context>: %w", ErrXxx)` — ensures `errors.Is` works through the chain (NFR-R1, NFR-R2)
- Named return + defer/recover is the canonical Go pattern for panic-safe functions; the `err` named return must be set inside the deferred function for this to work correctly
- Panic recovery log: `slog.ErrorContext(ctx, ...)` — never log the panic value if it could contain sensitive data; log `fmt.Sprintf("%v", rec)` to avoid accidental structured-logging of complex types
- The `recover()` in the defer only activates when a panic has fired — it does NOT suppress normal error returns
- `ErrNoEligibleProvider` is the panic recovery return value (NFR-R1) — the gateway will map it to 503
- `ErrInvalidWorkloadClass` maps to 400 Bad Request (Story 2.4) — make sure the profile name is in the error message for operator debugging
- After this story, all error cases for `Resolve` are handled: unknown profile → 400, all exhausted → 503, panic → 503 (no crash)

### Project Structure Notes

- File: `internal/routing/router.go` (extending `Resolve` from Story 2.2)
- Named return is only needed on `Resolve` — do not apply it to other methods unless required

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#FR7, NFR-R1, NFR-R2, NFR-R3] — Sentinel errors; no silent routing; panic containment
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — "Panic recovery in Resolve via named-return defer/recover"; error wrapping convention
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.3] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
