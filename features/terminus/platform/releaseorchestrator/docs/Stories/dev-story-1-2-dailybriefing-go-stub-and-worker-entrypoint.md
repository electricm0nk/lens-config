# Story 1.2: DailyBriefing Go Stub and Worker Entrypoint

Status: ready-for-dev

## Story

As a platform developer,
I want the `DailyBriefingWorkflow` Go stub and `cmd/worker/main.go` entrypoint wired together so that the worker binary compiles and registers on the `terminus-platform` task queue,
so that the Go worker is a valid replacement for the Node.js worker at the binary level.

## Acceptance Criteria

1. `internal/workflows/daily_briefing.go` declares `DailyBriefingWorkflow` accepting `types.ModulePayload[DailyBriefingSchema]` and performing the same payload validation (date + timezone required) as the TypeScript original — no other logic
2. `internal/workflows/daily_briefing_test.go` covers: valid payload passes, missing date returns error, missing timezone returns error
3. `internal/activities/briefing.go` declares `FetchBriefingData` activity returning a stub `ModulePayload` with empty sources — matching TypeScript semantics
4. `internal/activities/briefing_test.go` covers: happy path returns non-nil result; error path wraps as `temporal.NewNonRetryableApplicationError`
5. `cmd/worker/main.go` reads `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE` from `os.Getenv` with defaults matching the TypeScript original
6. `cmd/worker/main.go` registers `DailyBriefingWorkflow` and `briefingActivities` on the worker
7. `go build ./cmd/worker/` produces a binary without errors
8. `go test ./...` passes with ≥ 80% line coverage
9. No `time.Now()`, `os.Getenv`, or `http` calls inside `internal/workflows/` files
10. All activity logging uses `slog.Info`/`slog.Error` with structured key/value pairs

## Tasks / Subtasks

- [ ] Add `go.temporal.io/sdk` dependency: `go get go.temporal.io/sdk` (AC: 7)
- [ ] Write `internal/workflows/daily_briefing_test.go` FIRST (TDD) — 3 cases (AC: 2)
- [ ] Implement `internal/workflows/daily_briefing.go` with `DailyBriefingSchema` local struct and workflow validation (AC: 1, 9)
- [ ] Write `internal/activities/briefing_test.go` FIRST (TDD) — 2 cases (AC: 4)
- [ ] Implement `internal/activities/briefing.go` with `FetchBriefingData` stub (AC: 3, 10)
- [ ] Implement `cmd/worker/main.go` reading env vars, creating client, registering workflow+activity, starting worker (AC: 5, 6)
- [ ] Verify `go build ./cmd/worker/` passes (AC: 7)
- [ ] Verify `go test ./...` ≥ 80% coverage (AC: 8)

## Dev Notes

### Dependencies

Add to `go.mod` via:
```bash
go get go.temporal.io/sdk
```

This will also add `go.temporal.io/api` and transitive dependencies.

### DailyBriefingSchema

Define locally in `internal/workflows/daily_briefing.go` — NOT in `internal/types/`. It's workflow-specific, not a shared domain type:

```go
package workflows

import (
    "go.temporal.io/sdk/workflow"
    "github.com/electricm0nk/terminus-platform/internal/types"
)

// DailyBriefingSchema is the workflow-specific payload schema for DailyBriefingWorkflow.
type DailyBriefingSchema struct {
    Date     string
    Timezone string
}

// DailyBriefingWorkflow validates the payload and returns.
// This is a no-op stub — same behaviour as the TypeScript original.
func DailyBriefingWorkflow(ctx workflow.Context, payload types.ModulePayload[DailyBriefingSchema]) error {
    if payload.Schema.Date == "" {
        return temporal.NewNonRetryableApplicationError("date is required", "ValidationError", nil)
    }
    if payload.Schema.Timezone == "" {
        return temporal.NewNonRetryableApplicationError("timezone is required", "ValidationError", nil)
    }
    return nil
}
```

**Critical:** Workflows must be deterministic. No `time.Now()`, `os.Getenv`, `http`, or any I/O inside workflow functions. Temporal SDK panics on violations in replay.

### FetchBriefingData Activity Stub

```go
package activities

import (
    "context"
    "log/slog"
    "github.com/electricm0nk/terminus-platform/internal/types"
)

// BriefingActivities holds the briefing activity implementations.
type BriefingActivities struct{}

// FetchBriefingData is a stub activity that returns an empty ModulePayload.
func (a *BriefingActivities) FetchBriefingData(ctx context.Context) (types.ModulePayload[any], error) {
    slog.Info("fetch-briefing-data called")
    return types.ModulePayload[any]{}, nil
}
```

### Worker Entrypoint

```go
package main

import (
    "log/slog"
    "os"

    "go.temporal.io/sdk/client"
    "go.temporal.io/sdk/worker"
    "github.com/electricm0nk/terminus-platform/internal/activities"
    "github.com/electricm0nk/terminus-platform/internal/workflows"
)

func main() {
    address   := getenv("TEMPORAL_ADDRESS", "localhost:7233")
    namespace := getenv("TEMPORAL_NAMESPACE", "default")
    taskQueue := getenv("TEMPORAL_TASK_QUEUE", "terminus-platform")

    c, err := client.Dial(client.Options{
        HostPort:  address,
        Namespace: namespace,
    })
    if err != nil {
        slog.Error("unable to create temporal client", "error", err)
        os.Exit(1)
    }
    defer c.Close()

    w := worker.New(c, taskQueue, worker.Options{})
    w.RegisterWorkflow(workflows.DailyBriefingWorkflow)
    w.RegisterActivity(&activities.BriefingActivities{})

    if err := w.Run(worker.InterruptCh()); err != nil {
        slog.Error("worker exited", "error", err)
        os.Exit(1)
    }
}

func getenv(key, fallback string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return fallback
}
```

**Note:** `TEMPORAL_TASK_QUEUE` default must be `"terminus-platform"` — this is the locked task queue name per architecture.

### Logging Rules

- All `slog` calls use structured pairs: `slog.Info("message", "key", value)`
- Never use `fmt.Println` or `fmt.Printf` in non-test code
- Never log inside workflow functions (determinism violation)

### Test Patterns

For workflow tests use `testsuite.WorkflowTestSuite` from `go.temporal.io/sdk/testsuite`. For activity tests, plain `context.Background()` is sufficient for stubs.

### Project Structure After This Story

```
cmd/
  worker/
    main.go               # NEW: Temporal worker entrypoint
internal/
  workflows/
    daily_briefing.go     # NEW: DailyBriefingWorkflow stub
    daily_briefing_test.go # NEW (TDD: written first)
  activities/
    briefing.go           # NEW: FetchBriefingData stub
    briefing_test.go      # NEW (TDD: written first)
  types/                  # FROM Story 1.1
go.mod                    # UPDATED: go.temporal.io/sdk added
go.sum                    # UPDATED
```

### References

- [Architecture: Workflow Trigger](docs/terminus/platform/releaseorchestrator/architecture.md#workflow-trigger)
- [Architecture: Anti-Patterns Forbidden](docs/terminus/platform/releaseorchestrator/architecture.md#anti-patterns-forbidden)
- [Architecture: Process Patterns](docs/terminus/platform/releaseorchestrator/architecture.md#process-patterns)
- [Architecture: Naming Patterns](docs/terminus/platform/releaseorchestrator/architecture.md#naming-patterns)
- [Epics: Story 1.2](docs/terminus/platform/releaseorchestrator/epics.md#story-12-dailybriefing-go-stub-and-worker-entrypoint)
- TypeScript original for semantic reference: `services/temporal/src/workflows/dailyBriefingWorkflow.ts`

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
