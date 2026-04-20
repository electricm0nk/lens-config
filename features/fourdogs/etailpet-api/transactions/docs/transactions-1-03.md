# Story: transactions-1-03

## Context

**Feature:** transactions
**Sprint:** 1 — Core Implementation
**Priority:** medium
**Estimate:** S

## User Story

As a developer, I want comprehensive unit tests for the `internal/etailpet/` package, so that
auth injection, retry logic, error classification, and config validation are verified without
network calls.

## Acceptance Criteria

- [ ] `internal/etailpet/client_test.go` present with tests covering:
  - Successful trigger: mock HTTP 200 → `Trigger()` returns nil; `trigger_success` logged
  - Auth header injection: mock verifies correct header name and value; API key never logged
  - Retry on 5xx: mock returns 500 twice, then 200 → `Trigger()` returns nil; 2 `trigger_backoff` events logged
  - Retry exhaustion: mock returns 500 three times → `Trigger()` returns error; `trigger_exhausted` logged
  - Non-retryable 4xx (e.g. 400): mock returns 400 → `Trigger()` returns error immediately; no backoff events
  - 429 rate limit: mock returns 429 → treated as retryable (same as 5xx)
  - Network error: mock transport returns `io.EOF` → `Trigger()` returns error; retry applied
- [ ] `internal/etailpet/config_test.go` present with tests covering:
  - All required env vars present → config loads without error
  - Missing required env var → `MustLoadConfig()` panics with descriptive message
  - `TRIGGER_INTERVAL_HOURS` valid integer → parsed correctly
  - `TRIGGER_INTERVAL_HOURS` invalid value → config load returns error
  - `TRIGGER_ENABLED=false` → `cfg.Enabled == false`
- [ ] All tests pass with `go test ./internal/etailpet/... -v -count=1`
- [ ] No real HTTP calls in any test (mock transport only)
- [ ] Test file uses `t.Setenv()` for env var isolation (no global state leakage between tests)

## Technical Notes

**Mock HTTP transport pattern:**
```go
type mockTransport struct {
    responses []*http.Response
    errors    []error
    callCount int
}

func (m *mockTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    i := m.callCount
    m.callCount++
    if i < len(m.errors) && m.errors[i] != nil {
        return nil, m.errors[i]
    }
    return m.responses[i], nil
}
```

Inject via `client.httpClient = &http.Client{Transport: &mockTransport{...}}`.

**Log capture for assertion:** Use `slog.New(slog.NewJSONHandler(&logBuf, nil))` and assert
that expected structured events appear in `logBuf` output.

## Dependencies

**Blocked by:** transactions-1-01 (package must exist to test)
**Blocks:** nothing (tests are parallel-safe)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

None.
