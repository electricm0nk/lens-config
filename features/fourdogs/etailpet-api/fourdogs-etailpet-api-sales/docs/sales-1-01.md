# Story: sales-1-01

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 1 — LegacyClient and Sales Trigger Binary
**Priority:** high
**Estimate:** M

## User Story

As a developer, I want a `LegacyClient` in the `internal/etailpet/` package that encapsulates the
`www.etailpet.com` query-param OAuth2 token flow and the sales-line trigger GET, so that the
auth and trigger logic are testable and isolated from the scheduler binary.

## Acceptance Criteria

- [ ] `internal/etailpet/legacy_client.go` implements `LegacyClient` struct with `TriggerSalesLineExport(ctx context.Context, startDate, endDate time.Time) error` method
- [ ] Token request uses query-param OAuth2 form (NOT JSON body): `POST /{schema}/oauth2/token/?grant_type=client_credentials&client_secret=...&client_id=...`
- [ ] Token extracted from `access_token` field; never logged
- [ ] Trigger GET uses `Authorization: Bearer {token}`, `report_type=sales-line`, `file_format=xlsx`, `start_date`/`end_date` in `YYYY-MM-DD` format
- [ ] `LegacyConfig` struct in `internal/etailpet/legacy_config.go` validates required env vars at startup
- [ ] `http.Client{Timeout: 30s}` applied to both token request and trigger GET
- [ ] Retry with exponential backoff: 3 attempts, initial backoff 5s, max backoff 60s; 4xx (non-429) classified as non-retryable
- [ ] Structured log events emitted via `log/slog`:
  - `trigger_starting` — `{report_type, interval}`
  - `trigger_invoked` — `{report_type, start_date, end_date, timestamp}`
  - `trigger_success` — `{status_code, latency_ms}`
  - `trigger_failure` — `{status_code, error, attempt}`
  - `trigger_backoff` — `{attempt, delay_ms}`
  - `trigger_exhausted` — `{total_attempts, final_error}`
- [ ] `LegacyClient` is distinct from the existing `Client` (pos domain); no shared state between the two

## Technical Notes

**Package additions:**
```
internal/etailpet/
├── legacy_client.go      ← new: LegacyClient struct, TriggerSalesLineExport()
├── legacy_config.go      ← new: LegacyConfig struct, env var parsing
├── client.go             ← existing (pos domain); no changes required
└── config.go             ← existing; no changes required
```

**Token request shape** (confirmed by sales-0-01 spike):
```go
tokenURL := fmt.Sprintf(
    "%s/%s/oauth2/token/?grant_type=client_credentials&client_secret=%s&client_id=%s",
    cfg.LegacyBaseURL, cfg.SchemaName,
    url.QueryEscape(cfg.ClientSecret),
    url.QueryEscape(cfg.ClientID),
)
req, _ := http.NewRequestWithContext(ctx, http.MethodPost, tokenURL, nil)
```

Token response shape:
```json
{"access_token": "...", "token_type": "Bearer", "expires_in": 36000}
```

**Trigger request shape:**
```go
triggerURL := fmt.Sprintf("%s/%s/api/v1/transaction-report/", cfg.LegacyBaseURL, cfg.SchemaName)
q := url.Values{}
q.Set("report_type", "sales-line")
q.Set("file_format", "xlsx")
q.Set("start_date", startDate.Format("2006-01-02"))  // YYYY-MM-DD
q.Set("end_date", endDate.Format("2006-01-02"))
req, _ := http.NewRequestWithContext(ctx, http.MethodGet, triggerURL+"?"+q.Encode(), nil)
req.Header.Set("Authorization", "Bearer "+token)
req.Header.Set("Content-Type", "application/json")
```

**Date format note:** `www.etailpet.com` uses `YYYY-MM-DD` (`2006-01-02` in Go). The `pos` client
uses `MM/DD/YYYY` (`01/02/2006`). Both use `ETAILPET_TIMEZONE` for window anchoring — shared
utility function acceptable.

**LegacyConfig env vars:**

Vault-injected (ESO, same Vault path as pos):
- `ETAILPET_CLIENT_ID`
- `ETAILPET_CLIENT_SECRET`

Helm values (non-sensitive):
- `ETAILPET_LEGACY_BASE_URL` (default `https://www.etailpet.com`)
- `ETAILPET_SCHEMA_NAME` (default `fourdogspetsupplies`)
- `ETAILPET_TIMEZONE` (default `America/Los_Angeles`)
- `TRIGGER_LOOKBACK_DAYS` (default `7`)

## Dependencies

**Blocked by:** sales-0-01 (token form and trigger behavior must be confirmed before implementation)
**Blocks:** sales-1-02, sales-1-03

## Definition of Done

- [ ] `go build ./internal/etailpet/...` passes
- [ ] `go vet ./internal/etailpet/...` passes
- [ ] No credentials in code or committed files
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
