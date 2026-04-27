# Story: sales-1-02

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 1 — LegacyClient and Sales Trigger Binary
**Priority:** high
**Estimate:** M

## User Story

As an operator, I want a `cmd/etailpet-sales-trigger` binary that runs as a standalone k8s
Deployment and triggers the eTailPet sales-line export on a configurable daily schedule, so that
the export is automated without manual portal access.

## Acceptance Criteria

- [ ] `cmd/etailpet-sales-trigger/main.go` builds as a standalone binary
- [ ] `LegacyConfig` loaded and validated at startup; process exits non-zero if required env vars missing
- [ ] `time.Ticker` fires on `TRIGGER_INTERVAL_HOURS` schedule (default: 24); first tick fires immediately at startup (same pattern as transactions worker)
- [ ] `TRIGGER_ENABLED=false` suppresses trigger and emits `trigger_disabled` log event; process continues running (does not exit)
- [ ] `--dry-run` flag suppresses HTTP calls and emits `trigger_dry_run` log event
- [ ] SIGTERM received: in-progress trigger cycle completes; process exits cleanly (no force-kill)
- [ ] `trigger_starting` structured log event emitted at startup with `{interval_hours, lookback_days, report_type}`
- [ ] Binary accepts no inbound HTTP connections (no HTTP server); no `/healthz` port exposed

## Technical Notes

**Binary structure:**
```
cmd/etailpet-sales-trigger/
└── main.go      — startup, config load, ticker loop, signal handling
```

**Main loop pattern** (matches transactions worker):
```go
func main() {
    cfg := etailpet.LoadLegacyConfig()
    client := etailpet.NewLegacyClient(cfg)

    slog.Info("trigger_starting", "interval_hours", cfg.IntervalHours, "report_type", "sales-line")

    // Fire immediately at startup
    runCycle(ctx, client, cfg, "startup")

    ticker := time.NewTicker(time.Duration(cfg.IntervalHours) * time.Hour)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            runCycle(ctx, client, cfg, "scheduled")
        case <-ctx.Done():
            slog.Info("trigger_shutdown")
            return
        }
    }
}
```

**Signal handling:**
```go
ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
defer cancel()
```

**Date window** (rolling lookback from today):
```go
tz, _ := time.LoadLocation(cfg.Timezone)
now := time.Now().In(tz)
endDate := now.Truncate(24 * time.Hour)
startDate := endDate.AddDate(0, 0, -cfg.LookbackDays)
```

**`--dry-run` flag:**
```go
var dryRun bool
flag.BoolVar(&dryRun, "dry-run", false, "Log trigger parameters without making API calls")
flag.Parse()
```

The `--dry-run` flag is passed to `runCycle`; if true, log `trigger_dry_run` and return without
calling `LegacyClient`.

## Dependencies

**Blocked by:** sales-1-01 (LegacyClient must exist)
**Blocks:** sales-1-04

## Definition of Done

- [ ] `go build ./cmd/etailpet-sales-trigger/...` passes
- [ ] `go vet ./cmd/etailpet-sales-trigger/...` passes
- [ ] `--dry-run` flag exits 0 with expected log output
- [ ] `TRIGGER_ENABLED=false` suppresses trigger without process exit
- [ ] SIGTERM handling confirmed (manual test or unit test with context cancel)
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
