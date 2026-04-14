# Story 4.3: HealthCheck and logging tests

Status: ready-for-dev

## Story

As a **developer**,
I want `router_test.go` to include tests for `HealthCheck` and log output verification,
So that health correctness and PII-safety are regression-tested.

## Acceptance Criteria

1. **Given** `router_test.go` tests `HealthCheck`  
   **When** `go test -race ./internal/routing/...` is run  
   **Then** test cases cover: healthy state (no exhaustion), partially exhausted (one provider), fully exhausted (all providers)

2. **And** a test verifies that a log handler capturing `slog` records contains no credential or prompt content strings across all `Resolve` call paths

3. **And** zero race conditions are detected

## Tasks / Subtasks

- [ ] Extend `router_test.go` with `TestHealthCheck` table-driven tests (AC: 1, 3)
  - [ ] Healthy state: 2 providers, neither exhausted → `AllExhausted: false`, `Degraded: false`
  - [ ] Partial exhaustion: openai exhausted, ollama not → `AllExhausted: false`, openai `BudgetExhausted: true`
  - [ ] Full exhaustion: all providers exhausted → `AllExhausted: true`, `Degraded: true`
- [ ] Write `TestResolve_LogPIISafety` using a capturing `slog.Handler` (AC: 2)
  - [ ] Install a custom capturing handler via `slog.SetDefault(slog.New(capturingHandler))`
  - [ ] Call `Resolve` for: success path, degraded path, no-provider path, panic path
  - [ ] For each captured log record: assert no field value matches a known credential placeholder (`"test-api-key"`, `"secret"`) or prompt content
  - [ ] Restore default handler in `t.Cleanup`
- [ ] Run `go test -race ./internal/routing/...` — confirm all pass, 0 race findings (AC: 3)

## Dev Notes

- Capturing `slog.Handler` implementation sketch:
  ```go
  type capturingHandler struct {
      mu      sync.Mutex
      records []slog.Record
  }
  func (h *capturingHandler) Enabled(_ context.Context, _ slog.Level) bool { return true }
  func (h *capturingHandler) Handle(_ context.Context, r slog.Record) error {
      h.mu.Lock(); defer h.mu.Unlock()
      h.records = append(h.records, r)
      return nil
  }
  func (h *capturingHandler) WithAttrs(attrs []slog.Attr) slog.Handler { return h }
  func (h *capturingHandler) WithGroup(name string) slog.Handler { return h }
  ```
- For PII safety assertion: iterate each `slog.Record`'s attributes and check that no attribute value string-contains the forbidden test credential values
- `slog.SetDefault` is global — use `t.Cleanup` to restore the original default handler after each test
- For the `HealthCheck` USD values: when testing partial exhaustion, pre-accumulate token spend via `BudgetTracker.AddSpend` and verify `BudgetUsedUSD` reflects it accurately (roughly, allowing for float precision)
- `TestHealthCheck_Concurrent`: optional but recommended — call `HealthCheck` while 10 goroutines call `AddSpend` simultaneously; verify no race

### Project Structure Notes

- File: `internal/routing/router_test.go` (extending existing test file from Story 2.5)
- The `capturingHandler` can be defined in a `_test.go` helper file or inline in `router_test.go`

### References

- [Source: docs/terminus/inference/provider-routing/prd.md#NFR-S1, NFR-S2] — No credentials or PII in logs
- [Source: docs/terminus/inference/provider-routing/architecture.md#Additional Requirements] — Logging allow-list
- [Source: docs/terminus/inference/provider-routing/epics.md#Story 4.3] — All AC scenarios

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
