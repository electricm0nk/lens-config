# Story: transactions-1-02

## Context

**Feature:** transactions
**Sprint:** 1 — Core Implementation
**Priority:** high
**Estimate:** M

## User Story

As an operator, I want a `cmd/etailpet-trigger` binary that runs on a daily schedule and fires
the eTailPet API trigger, so that the data export is automated without manual intervention.

## Acceptance Criteria

- [ ] `cmd/etailpet-trigger/main.go` compiles and runs as a standalone binary
- [ ] Binary reads all config from environment variables; fails fast with clear error on missing required vars
- [ ] `--dry-run` flag: logs trigger intent (`trigger_dry_run` event) without making any HTTP call; exits 0
- [ ] `time.Ticker` fires on configured interval (default: 24h); interval configurable via `TRIGGER_INTERVAL_HOURS` env var
- [ ] `TRIGGER_ENABLED` env var: if `false`, binary logs `trigger_disabled` and enters idle loop without ever calling the API
- [ ] Graceful shutdown: SIGTERM / SIGINT caught; in-flight retry waits up to 10s for completion then exits
- [ ] Startup log: `trigger_starting` event with `schedule_interval`, `dry_run`, `enabled` fields
- [ ] Each tick logs `trigger_scheduled` with `next_run_at` timestamp before calling `etailpet.Client.Trigger()`
- [ ] Exit code 0 on clean shutdown; exit code 1 on fatal startup error

## Technical Notes

**Binary entry point (`cmd/etailpet-trigger/main.go`):**
```go
func main() {
    cfg := etailpet.MustLoadConfig()  // fails fast on missing env vars
    client := etailpet.NewClient(cfg)

    dryRun := flag.Bool("dry-run", false, "log intent without calling API")
    flag.Parse()

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
    defer stop()

    ticker := time.NewTicker(cfg.TriggerInterval)
    defer ticker.Stop()

    slog.Info("trigger_starting",
        "schedule_interval", cfg.TriggerInterval.String(),
        "dry_run", *dryRun,
        "enabled", cfg.Enabled)

    for {
        select {
        case t := <-ticker.C:
            slog.Info("trigger_scheduled", "fired_at", t)
            if !cfg.Enabled {
                slog.Info("trigger_disabled")
                continue
            }
            if *dryRun {
                slog.Info("trigger_dry_run")
                continue
            }
            if err := client.Trigger(ctx); err != nil {
                slog.Error("trigger_cycle_error", "error", err.Error())
            }
        case <-ctx.Done():
            slog.Info("trigger_shutdown")
            return
        }
    }
}
```

**Environment variables:**
| Var | Default | Required |
|-----|---------|---------|
| `ETAILPET_API_KEY` | — | Yes (or auth equivalent per spike) |
| `ETAILPET_TRIGGER_URL` | — | Yes |
| `TRIGGER_INTERVAL_HOURS` | `24` | No |
| `TRIGGER_ENABLED` | `true` | No |

**Container:** Binary is compiled into the existing `fourdogs-central` Docker image or a new
`fourdogs-etailpet-trigger` image (decision: use a separate image per Helm chart convention to
keep deployments independent). Dockerfile: `FROM gcr.io/distroless/static-debian12`.

## Dependencies

**Blocked by:** transactions-1-01 (`internal/etailpet/` package must exist)
**Blocks:** transactions-2-01 (Helm chart wraps this binary)

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
