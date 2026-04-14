# Story 4.3: HealthCheck and logging tests

**Initiative:** terminus-inference-provider-routing
**Epic:** 4 — Operator can observe routing health
**Status:** review

## Story

As a **developer**,
I want `router_test.go` to include tests for `HealthCheck` and log output verification,
So that health correctness and PII-safety are regression-tested.

## Acceptance Criteria

**Given** `router_test.go` tests `HealthCheck`
**When** `go test -race ./internal/routing/...` is run
**Then** test cases cover:
- Healthy state — no exhaustion, `AllExhausted: false`, `Degraded: false`
- Partially exhausted — one of two providers exhausted, `AllExhausted: false`, `Degraded: true`
- Fully exhausted — all providers exhausted, `AllExhausted: true`, `Degraded: true`

**And** a test verifies that a `slog` record capture across all `Resolve` call paths contains no credential or prompt content strings
**And** zero race conditions are detected

## Technical Notes

- Target repo: `terminus-inference-gateway`
- Add to existing `internal/routing/router_test.go`
- For PII-safety log test: use `slog/slogtest` or a custom `slog.Handler` that captures records; assert none contain known credential strings
- PII-safety test forbidden substrings: any actual key value, any literal from the test request body
- HealthCheck tests can use direct struct manipulation (unexported budget state) — white-box package

## Dependencies

- Story 4.1 (logging implemented)
- Story 4.2 (HealthCheck implemented)
