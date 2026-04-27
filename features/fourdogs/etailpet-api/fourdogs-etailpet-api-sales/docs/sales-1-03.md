# Story: sales-1-03

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 1 — LegacyClient and Sales Trigger Binary
**Priority:** medium
**Estimate:** S

## User Story

As a developer, I want unit tests for `LegacyClient` using a mock HTTP transport, so that the
token flow, trigger request, retry logic, date formatting, and error handling are verified without
network calls.

## Acceptance Criteria

- [ ] `internal/etailpet/legacy_client_test.go` exists; tests run with `go test ./internal/etailpet/...`
- [ ] Happy path: token POST returns valid `access_token`; trigger GET returns `200 OK`; `TriggerSalesLineExport` returns nil
- [ ] Auth failure: token POST returns `401`; error returned; no trigger GET attempted
- [ ] Rate limit: trigger GET returns `429`; retry fires with backoff; all 3 attempts exhausted → `trigger_exhausted` logged; error returned
- [ ] Non-retryable 4xx: trigger GET returns `400`; no retry; error returned immediately
- [ ] Server error: trigger GET returns `500`; retry fires; all 3 attempts fail; error returned
- [ ] `start_date` / `end_date` formatted as `YYYY-MM-DD` in trigger GET query params (NOT `MM/DD/YYYY`)
- [ ] `http.Client.Timeout` is respected: mock slow response → `TriggerSalesLineExport` returns timeout error
- [ ] `LegacyConfig` validation: missing `ETAILPET_CLIENT_ID` or `ETAILPET_CLIENT_SECRET` causes startup failure with clear error

## Technical Notes

**Mock transport pattern** (same as transactions tests):
```go
type mockTransport struct {
    roundTripFn func(req *http.Request) (*http.Response, error)
}
func (m *mockTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    return m.roundTripFn(req)
}
```

**Token response helper:**
```go
func tokenResponse(token string) *http.Response {
    body := fmt.Sprintf(`{"access_token":%q,"token_type":"Bearer","expires_in":36000}`, token)
    return &http.Response{
        StatusCode: 200,
        Body:       io.NopCloser(strings.NewReader(body)),
    }
}
```

**Date format test:**
```go
start := time.Date(2026, 4, 20, 0, 0, 0, 0, time.UTC)
end := time.Date(2026, 4, 27, 0, 0, 0, 0, time.UTC)
// verify query params contain: start_date=2026-04-20, end_date=2026-04-27
// verify they do NOT contain MM/DD/YYYY format
```

**No network calls:** All tests use the mock transport. `go test -run ./...` must pass without
internet access.

## Dependencies

**Blocked by:** sales-1-01
**Blocks:** sales-1-04

## Definition of Done

- [ ] `go test ./internal/etailpet/...` passes with no test failures
- [ ] `go vet ./internal/etailpet/...` passes
- [ ] Test coverage for LegacyClient happy path and all error paths
- [ ] Date format tests explicitly assert YYYY-MM-DD output
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
