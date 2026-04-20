# Story: transactions-1-01

## Context

**Feature:** transactions
**Sprint:** 1 — Core Implementation
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want an `internal/etailpet/` Go package with a trigger client, so that the
API call is encapsulated, testable, and isolated from the scheduler binary.

## Acceptance Criteria

- [ ] `internal/etailpet/client.go` implements `Client` struct with `Trigger(ctx context.Context) error` method
- [ ] Client reads auth credential from config (not hardcoded); credential never appears in logs
- [ ] Client sends the correct HTTP method, URL, headers, and body per spike findings (transactions-0-01)
- [ ] Client returns a structured error type distinguishing: auth failure (401/403), rate limit (429), server error (5xx), network error
- [ ] Client implements retry with exponential backoff: 3 attempts, initial backoff 5s, max backoff 60s
- [ ] Client logs `trigger_attempt`, `trigger_success`, `trigger_failure`, `trigger_backoff`, `trigger_exhausted` events via `log/slog` (JSON, structured)
- [ ] Job ID / correlation token logged if returned by API (field: `job_id`; omitted if absent)
- [ ] `internal/etailpet/config.go` validates required env vars at startup; fails fast with clear error if missing
- [ ] All public types and functions have Go doc comments

## Technical Notes

**Package structure:**
```
internal/etailpet/
├── client.go       — Client struct, Trigger(), retry loop
├── config.go       — Config struct, env var parsing, validation
└── client_test.go  — unit tests (see story transactions-1-03)
```

**Auth pattern** (exact implementation depends on spike findings from transactions-0-01):
- API key header: `cfg.APIKey` injected into `Authorization: Bearer <key>` or `X-API-Key: <key>`
- OAuth: token source wrapping `golang.org/x/oauth2`; reuse refresh token from k8s Secret
- Basic auth: `req.SetBasicAuth(cfg.Username, cfg.Password)`

**Retry logic:**
```go
backoff := 5 * time.Second
for attempt := 1; attempt <= 3; attempt++ {
    err = c.doRequest(ctx)
    if err == nil { return nil }
    if isNonRetryable(err) { return err }  // 4xx other than 429
    slog.Warn("trigger_backoff", "attempt", attempt, "backoff_ms", backoff.Milliseconds())
    time.Sleep(backoff)
    backoff = min(backoff*2, 60*time.Second)
}
slog.Error("trigger_exhausted", "total_attempts", 3, "final_error", err)
return err
```

**Log event schema:**
```go
slog.Info("trigger_success", "status_code", resp.StatusCode, "latency_ms", latencyMs, "job_id", jobID)
slog.Error("trigger_failure", "status_code", resp.StatusCode, "error", err.Error(), "attempt", attempt)
```

**Never log:** API key, password, Authorization header value, response body in full (may contain sensitive data).

**Config env vars** (exact names depend on spike findings):
- `ETAILPET_API_KEY` — API credential (or `ETAILPET_USERNAME` + `ETAILPET_PASSWORD` for basic auth)
- `ETAILPET_TRIGGER_URL` — trigger endpoint URL (allows overriding for staging/prod)

Follow the `fourdogs-central` config pattern in `internal/config/config.go`.

## Dependencies

**Blocked by:** transactions-0-01 (auth type and endpoint must be confirmed)
**Blocks:** transactions-1-02 (binary imports this package)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

None — resolved by transactions-0-01.
