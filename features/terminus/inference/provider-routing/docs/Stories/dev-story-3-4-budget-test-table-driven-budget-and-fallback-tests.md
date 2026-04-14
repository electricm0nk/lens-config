# Story 3.4: budget_test.go — table-driven budget and fallback tests

**Initiative:** terminus-inference-provider-routing
**Epic:** 3 — Batch workloads can't exhaust cloud credits
**Status:** review

## Story

As a **developer**,
I want `budget_test.go` to cover all `RecordUsage` and budget-exhaustion paths with table-driven tests and the race detector,
So that concurrency correctness and threshold behaviour are verified.

## Acceptance Criteria

**Given** `budget_test.go` uses table-driven tests in `package routing` (white-box)
**When** `go test -race ./internal/routing/...` is run
**Then** test cases cover:
- Token accumulation below threshold (not exhausted)
- Token accumulation crossing threshold (becomes exhausted)
- Unlimited provider (`daily_budget_usd: 0`) — never exhausted regardless of spend
- Nil-usage fallback estimate (both counts 0, len/4 used)
- Concurrent writes from 50 goroutines — no race conditions

**And** zero race conditions are detected in all cases

## Technical Notes

- Target repo: `terminus-inference-gateway`
- File: `internal/routing/budget_test.go`
- `package routing` — white-box testing
- Concurrent write test: 50 goroutines each calling `RecordUsage` with 10 tokens — verify final spend = 500, no race
- Use `testing.T.Parallel()` for the concurrent test case
- Threshold test: set limit to 100, accumulate 50+60=110 — verify `exhausted = true` after second call

## Dependencies

- Stories 3.2 and 3.3 (all budget/exhaustion paths implemented)
