# Story 2.4: ProvisionDatabase Activity Implementation

Status: backlog

## Story

As a platform developer,
I want the `ProvisionDatabase` activity implemented to trigger a named Semaphore task and poll it to completion so that the service's database is provisioned via Semaphore before ArgoCD sync proceeds,
so that the application starts with its schema already in place.

## Acceptance Criteria

1. `ProvisionDatabase` method on `*ReleaseActivities` fires a Semaphore task named `db-provision-{service}` via `POST /api/v1alpha1/project/{id}/tasks`
2. Method polls `GET /api/v1alpha1/project/{id}/tasks/{task_id}` every 10 seconds until status is `"finished"` or `"failed"`
3. `"finished"` → return `nil`
4. `"failed"` → return `temporal.NewNonRetryableApplicationError("db provision task failed", "ProvisionFailed", nil)`
5. The activity does NOT check `input.ProvisionDB` — that conditional belongs in the workflow (Story 2.2)
6. HTTP requests carry `Authorization: Bearer {SemaphoreAPIToken}` header
7. Non-2xx response from Semaphore returns a retryable `fmt.Errorf`
8. `slog.Info` called with structured key/value on task trigger and on each poll cycle; `slog.Error` on failure
9. `os.Getenv` is NOT called inside the method
10. `go test ./...` passes — at minimum one test for each terminal state (`"finished"`, `"failed"`)

**Pre-condition:** Story 2.3 is complete (pattern established; `triggerSemaphoreTask` and `pollSemaphoreTask` helpers exist).

## Tasks / Subtasks

- [ ] Implement `ProvisionDatabase` in `internal/activities/release.go` (AC: 1–9)
  - [ ] Task name: `db-provision-{input.ServiceName}` (AC: 1)
  - [ ] Re-use `triggerSemaphoreTask` and `pollSemaphoreTask` private helpers from Story 2.3 (AC: 2)
  - [ ] `"finished"` → nil (AC: 3)
  - [ ] `"failed"` → `NonRetryableApplicationError("db provision task failed", "ProvisionFailed", nil)` (AC: 4)
  - [ ] No `input.ProvisionDB` check inside the method (AC: 5)
  - [ ] Structured slog logging (AC: 8)
  - [ ] No `os.Getenv` in method body (AC: 9)
- [ ] Add test cases to `internal/activities/release_test.go` (AC: 10)
  - [ ] `db-provision` path: `finished` → nil
  - [ ] `db-provision` path: `failed` → NonRetryable error
  - [ ] Verify task name is `db-provision-{service}` not `seed-{service}-secrets`
- [ ] Run `go test ./...` (AC: 10)

## Dev Notes

### Implementation Pattern

This activity is structurally identical to `SeedSecrets` — same fire-and-poll pattern, same helpers, different task name and error type string.

```go
func (a *ReleaseActivities) ProvisionDatabase(ctx context.Context, input types.ReleaseInput) error {
    taskName := fmt.Sprintf("db-provision-%s", input.ServiceName)

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
            return temporal.NewNonRetryableApplicationError("db provision task failed", "ProvisionFailed", nil)
        }
    }
}
```

### Workflow Responsibility Boundary

This activity does NOT check `input.ProvisionDB`. The `ReleaseWorkflow` (Story 2.2) has:

```go
if input.ProvisionDB {
    workflow.ExecuteActivity(ctx, a.ProvisionDatabase, input)...
}
```

If `ProvisionDatabase` is called with `ProvisionDB: false`, that's a workflow bug — the activity trusts it was called intentionally.

### Test Coverage Note

The `httptest.NewServer` pattern from Story 2.3 is directly reusable here. In the test, verify that when the mock receives a POST, the request body contains `"task_name": "db-provision-fourdogs-central"` (not `"seed-fourdogs-central-secrets"`).

### References

- [Architecture: ProvisionDatabase Activity](docs/terminus/platform/releaseorchestrator/architecture.md#provisiondatabase)
- [Epics: Story 2.4](docs/terminus/platform/releaseorchestrator/epics.md#story-24-provisiondatabase-activity-implementation)
- Story 2.3 for shared helper pattern reference

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
