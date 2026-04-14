# Story 2.6: RunSmokeTest Activity Implementation

Status: backlog

## Story

As a platform developer,
I want the `RunSmokeTest` activity implemented to trigger a named Semaphore task and poll it to completion so that end-to-end deployment verification is automated as part of the release workflow,
so that the `ReleaseResult.SmokeTestPassed` flag accurately reflects whether the deployed service passed its smoke suite.

## Acceptance Criteria

1. `RunSmokeTest` method on `*ReleaseActivities` fires a Semaphore task named `verify-{service}-deploy` via `POST /api/v1alpha1/project/{id}/tasks`
2. Method polls `GET /api/v1alpha1/project/{id}/tasks/{task_id}` every 10 seconds until status is `"finished"` or `"failed"`
3. `"finished"` → return `nil`
4. `"failed"` → return `temporal.NewNonRetryableApplicationError("smoke test failed", "SmokeTestFailed", nil)`
5. HTTP requests carry `Authorization: Bearer {SemaphoreAPIToken}` header
6. Non-2xx response from Semaphore returns a retryable `fmt.Errorf`
7. `slog.Info` called with structured key/value on task trigger and on each poll cycle
8. `os.Getenv` is NOT called inside the method
9. `go test ./...` passes **and overall module line coverage reaches ≥ 80%**
10. Completing this story closes the implementation gap left by the `RunSmokeTest` stub from Story 2.1

**Pre-condition:** Stories 2.3 and 2.4 complete — `triggerSemaphoreTask` and `pollSemaphoreTask` helpers are available. Story 2.5 complete (coverage baseline established).

## Tasks / Subtasks

- [ ] Implement `RunSmokeTest` in `internal/activities/release.go` (AC: 1–8)
  - [ ] Task name: `verify-{input.ServiceName}-deploy` (AC: 1)
  - [ ] Re-use `triggerSemaphoreTask` and `pollSemaphoreTask` helpers (AC: 2)
  - [ ] `"finished"` → nil (AC: 3)
  - [ ] `"failed"` → `NonRetryableApplicationError("smoke test failed", "SmokeTestFailed", nil)` (AC: 4)
  - [ ] `Authorization: Bearer {SemaphoreAPIToken}` header inherited from helpers (AC: 5)
  - [ ] Structured slog logging (AC: 7)
  - [ ] No `os.Getenv` (AC: 8)
- [ ] Add test cases to `internal/activities/release_test.go` (AC: 9)
  - [ ] `verify-{service}-deploy` path: `finished` → nil
  - [ ] `verify-{service}-deploy` path: `failed` → NonRetryable `SmokeTestFailed`
  - [ ] Check task name in POST body is `verify-fourdogs-central-deploy`
- [ ] Run `go test -coverprofile=coverage.out ./...` — verify ≥ 80% line coverage (AC: 9)
- [ ] If coverage is below 80%: identify uncovered branches and add targeted tests
- [ ] Run `go tool cover -html=coverage.out` to inspect coverage visually (optional — for finding gaps)

## Dev Notes

### Implementation Pattern

Identical fire-and-poll pattern as `SeedSecrets` and `ProvisionDatabase`:

```go
func (a *ReleaseActivities) RunSmokeTest(ctx context.Context, input types.ReleaseInput) error {
    taskName := fmt.Sprintf("verify-%s-deploy", input.ServiceName)

    taskID, err := a.triggerSemaphoreTask(ctx, taskName)
    if err != nil {
        return err
    }
    slog.Info("semaphore task triggered", "task_name", taskName, "task_id", taskID)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(10 * time.Second):
        }

        status, err := a.pollSemaphoreTask(ctx, taskID)
        if err != nil {
            return err
        }
        slog.Info("semaphore task poll", "task_name", taskName, "task_id", taskID, "status", status)

        switch status {
        case "finished":
            return nil
        case "failed":
            return temporal.NewNonRetryableApplicationError("smoke test failed", "SmokeTestFailed", nil)
        }
    }
}
```

### Coverage Gate — 80%

This story carries a coverage requirement for the **entire module**, not just this activity. Run:

```bash
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1
```

The final line shows total coverage percentage. Target: ≥ 80%.

If below 80%, check:
- `internal/workflows/` — ReleaseWorkflow ProvisionDB branch covered?
- `internal/activities/` — all three terminal states per activity covered?
- `internal/types/` — if there are constructor functions, are they called in tests?
- `cmd/worker/` — `main.go` is typically excluded from coverage; don't stress over it

Coverage must be checked here (not deferred to a separate story) because this is the last implementation story in Epic 2. After this, the only remaining work is Story 3.1 (k8s manifests — no Go code).

### Error Type Strings

Each activity has a distinct `errType` string for `NonRetryableApplicationError`. This allows Temporal's error handling and visibility to distinguish between failure modes:

| Activity | errType |
|---|---|
| SeedSecrets | `SemaphoreFailed` |
| ProvisionDatabase | `ProvisionFailed` |
| WaitForArgoCDSync | `SyncTimeout` |
| RunSmokeTest | `SmokeTestFailed` |

### References

- [Architecture: RunSmokeTest Activity](docs/terminus/platform/releaseorchestrator/architecture.md#runsmoketest)
- [Epics: Story 2.6](docs/terminus/platform/releaseorchestrator/epics.md#story-26-run-smoke-test-activity-implementation)
- Stories 2.3, 2.4 for shared helper pattern

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
