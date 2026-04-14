# Story 3.4: budget_test.go — table-driven budget and fallback tests

Status: ready-for-dev

## Story

As a **developer**,
I want `budget_test.go` to cover all `RecordUsage` and budget-exhaustion paths with table-driven tests and the race detector,
So that concurrency correctness and threshold behaviour are verified.

## Acceptance Criteria

1. **Given** `budget_test.go` uses table-driven tests in `package routing` (white-box)  
   **When** `go test -race ./internal/routing/...` is run  
   **Then** test cases cover: token accumulation below threshold, token accumulation crossing threshold, unlimited provider, nil-usage fallback estimate, concurrent writes from 50 goroutines

2. **And** zero race conditions are detected in all cases

## Tasks / Subtasks

- [ ] Create `internal/routing/budget_test.go` in `package routing` (AC: 1, 2)
- [ ] Write table-driven `TestBudgetTracker_AddSpend` covering (AC: 1):
  - [ ] Below threshold: 700 tokens of 1000 limit → `IsExhausted == false`, `tokenSpend == 700`
  - [ ] Crossing threshold: 700 → add 350 → total 1050 of 1000 limit → `IsExhausted == true`
  - [ ] Unlimited provider: add any tokens → `IsExhausted == false` always
- [ ] Write `TestRecordUsage_NilUsageFallback` (AC: 1):
  - [ ] Set up context with body length; call `RecordUsage` with 0 prompt + 0 completion tokens
  - [ ] Assert `AddSpend` received `len(body)/4` tokens (verify via `IsExhausted` threshold crossing)
  - [ ] Assert `slog.Warn` was emitted (capture via `slog.NewJSONHandler` or `slog.Default().Handler()`)
- [ ] Write `TestBudgetTracker_Concurrent` with 50-goroutine concurrent writers (AC: 1, 2):
  - [ ] Launch 50 goroutines, each calling `AddSpend` with 10 tokens simultaneously
  - [ ] After `WaitGroup.Wait()`: assert exhaustion state consistent (no partial writes)
  - [ ] Run under `-race` — zero data races
- [ ] Run `go test -race ./internal/routing/...` and confirm clean (AC: 2)

## Dev Notes

- `package routing` (white-box) for direct access to `BudgetTracker` internals for test assertions
- For `TestRecordUsage_NilUsageFallback`: pass body length via the context key defined in Story 3.2; construct the context with `context.WithValue(context.Background(), bodyLenContextKey, bodyLen)` — the context key type is unexported, so the test must be in `package routing`
- For slog capture: the easiest approach is a custom `slog.Handler` (implementing `Handle(context.Context, slog.Record) error`) that appends records for later assertion; alternatively check via `slog.SetDefault` with a capturing handler and `t.Cleanup` to restore
- `TestBudgetTracker_Concurrent`: verify final `tokenSpend` is exactly `50 * 10 = 500` tokens; any non-determinism in count indicates a race
- CI gate: `go test -race ./internal/routing/...` covers all test files in the package — run this from the gateway module root

### Project Structure Notes

- File: `internal/routing/budget_test.go`
- Test package: `package routing` (not `package routing_test`) — must have access to `bodyLenContextKey` and `providerBudget.tokenSpend`

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#NFR-P2, NFR-R3] — Non-blocking; no silent routing when exhausted
- [Source: docs/terminus/inference/provider-routing/architecture.md#Decision 1] — RWMutex; race-safe concurrent access
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 3.4] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
