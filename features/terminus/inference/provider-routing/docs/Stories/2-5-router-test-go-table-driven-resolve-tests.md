# Story 2.5: router_test.go — table-driven Resolve tests

Status: ready-for-dev

## Story

As a **developer**,
I want `router_test.go` to cover all `Resolve` paths with table-driven tests and the race detector,
So that routing logic is regression-safe and concurrent access is validated.

## Acceptance Criteria

1. **Given** `router_test.go` uses table-driven tests in `package routing` (white-box)  
   **When** `go test -race ./internal/routing/...` is run  
   **Then** all cases pass: happy path (first provider), fallback (second provider), invalid profile, all-unavailable, panic recovery

2. **And** mock providers implement the gateway `Provider` interface — no test-only interfaces are created

3. **And** all test cases use deterministic, non-network inputs

## Tasks / Subtasks

- [ ] Create `internal/routing/router_test.go` in `package routing` (white-box) (AC: 1)
- [ ] Write table-driven `TestResolve` covering all `Resolve` paths (AC: 1, 3):
  - [ ] Happy path: first provider eligible → returns `Provider: "openai"`, `Degraded: false`
  - [ ] Fallback: first provider exhausted, second eligible → returns second provider (from AC in Story 2.2)
  - [ ] Invalid profile: unknown workload class → `errors.Is(err, ErrInvalidWorkloadClass)`
  - [ ] All unavailable: all providers exhausted → `errors.Is(err, ErrNoEligibleProvider)`
  - [ ] Panic recovery: inject panic via a hook → `errors.Is(err, ErrNoEligibleProvider)`, no crash
- [ ] Write `TestResolve_Concurrent` with 100 goroutines all calling `Resolve` simultaneously (AC: 1, race)
- [ ] Confirm no test-only interfaces are created — use the real `BudgetTracker` stub (with exhaustion controllable via exported test helper or direct field access) (AC: 2)
- [ ] Run `go test -race ./internal/routing/...` and verify 0 failures and 0 race detections (AC: 1)

## Dev Notes

- `package routing` (white-box test package) gives direct access to unexported `router` struct for test setup; avoids the need for test-only interfaces
- For the exhaustion test cases: set `BudgetTracker` state directly (unexported field access within the package) rather than mocking the whole tracker
- Panic injection: the simplest approach is a test-only exported variable `var testPanicHook func()` that `Resolve` calls if non-nil (only compiled in test mode); alternatively, use a subdirectory with a `_test.go` build tag; confirm approach with existing gateway test conventions
- `TestResolve_Concurrent`: launch 100 goroutines with `sync.WaitGroup`, all calling `Resolve` with the same profile on the same `router` instance, verify all return consistent results with no data race
- All inputs must be deterministic (AC: 3): use fixed provider names, pre-loaded config structs, no random or time-dependent values
- CI gate: `go test -race ./internal/routing/...` — this is the gate for ALL routing tests, not just this story

### Project Structure Notes

- File: `internal/routing/router_test.go`
- Test package: `package routing` (not `package routing_test`) for white-box access
- The `BudgetTracker` stub from Story 2.2 must be controllable for exhaustion scenarios — ensure it exposes a way to set `exhausted = true` for specific providers in tests

### References

- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — `go test -race ./internal/routing/...` CI gate
- [Source: docs/terminus/inference/provider-routing/prd.md#NFR-P1] — < 1ms resolve (tests using in-memory state confirm this)
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 2.5] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
