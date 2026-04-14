# Story 2.3: SeedSecrets Activity Implementation

Status: backlog

## Story

As a platform developer,
I want the `SeedSecrets` activity implemented to trigger a named Semaphore task and poll it to completion so that service secrets are seeded into Vault before deployment proceeds,
so that ESO can sync those secrets to the Kubernetes namespace in time for ArgoCD sync.

## Acceptance Criteria

1. `SeedSecrets` method on `*ReleaseActivities` fires a Semaphore task named `seed-{service}-secrets` via `POST /api/v1alpha1/project/{id}/tasks`
2. Method polls `GET /api/v1alpha1/project/{id}/tasks/{task_id}` every 10 seconds until status is `"finished"` or `"failed"`
3. `"finished"` → return `nil`
4. `"failed"` → return `temporal.NewNonRetryableApplicationError("seed task failed", "SemaphoreFailed", nil)`
5. HTTP requests carry `Authorization: Bearer {SemaphoreAPIToken}` header
6. Non-2xx response from Semaphore returns a retryable `fmt.Errorf` (Temporal will retry the activity)
7. `slog.Info` called with structured key/value on task trigger and on each poll cycle; `slog.Error` on failure
8. `os.Getenv` is NOT called inside the method — all credentials come from struct fields
9. `go test ./...` passes — at minimum one table-driven test for each terminal state (`"finished"`, `"failed"`)

**Pre-condition:** Story 2.1 is complete. Vault path `secret/terminus/default/semaphore/api-token` must be seeded by operator (out of scope); ESO must have synced `release-semaphore-token` to the `temporal` namespace; `SEMAPHORE_API_TOKEN` and `SEMAPHORE_PROJECT_ID` must be set in the worker deployment.

## Tasks / Subtasks

- [ ] Implement `SeedSecrets` in `internal/activities/release.go` (AC: 1–8)
  - [ ] Build `POST /api/v1alpha1/project/{id}/tasks` request with JSON body `{"task_name": "seed-{service}-secrets"}`
  - [ ] Parse response to extract `task_id`
  - [ ] Poll loop: `GET /api/v1alpha1/project/{id}/tasks/{task_id}` every 10s
  - [ ] Parse poll response for `status` field
  - [ ] Handle `"finished"` → nil, `"failed"` → NonRetryableApplicationError
  - [ ] Handle non-2xx → retryable `fmt.Errorf`
  - [ ] Add `slog.Info` on trigger and each poll cycle (AC: 7)
  - [ ] Verify no `os.Getenv` in method body (AC: 8)
- [ ] Create or update `internal/activities/release_test.go` (AC: 9)
  - [ ] Use `httptest.NewServer` to mock Semaphore API
  - [ ] Table-driven test: `finished` path (returns nil), `failed` path (returns NonRetryable error)
  - [ ] Test that non-2xx returns retryable error (not NonRetryable)
- [ ] Run `go test ./...` (AC: 9)
- [ ] Manual smoke test: trigger workflow against staging Semaphore, observe task execution

## Dev Notes

### Semaphore API

Base URL comes from `SemaphoreBaseURL` — **you must add this field** to `ReleaseActivities` alongside `SemaphoreAPIToken` and `SemaphoreProjectID`. The existing stub didn't include a base URL field because it wasn't needed for the stub; add it now.

Wait — review Story 2.1's struct definition. If `SemaphoreBaseURL` was not added there, add it here and update `main.go` to inject `os.Getenv("SEMAPHORE_BASE_URL")`.

**POST trigger:**
```
POST {SemaphoreBaseURL}/api/v1alpha1/project/{SemaphoreProjectID}/tasks
Authorization: Bearer {SemaphoreAPIToken}
Content-Type: application/json

{"task_name": "seed-{input.ServiceName}-secrets"}
```

Response (202 Accepted):
```json
{"id": 42}
```

**GET poll:**
```
GET {SemaphoreBaseURL}/api/v1alpha1/project/{SemaphoreProjectID}/tasks/{task_id}
Authorization: Bearer {SemaphoreAPIToken}
```

Response:
```json
{"id": 42, "status": "running|finished|failed"}
```

### Implementation Pattern

```go
func (a *ReleaseActivities) SeedSecrets(ctx context.Context, input types.ReleaseInput) error {
    taskName := fmt.Sprintf("seed-%s-secrets", input.ServiceName)

    // Trigger task
    taskID, err := a.triggerSemaphoreTask(ctx, taskName)
    if err != nil {
        return err // retryable
    }
    slog.Info("semaphore task triggered", "task_name", taskName, "task_id", taskID)

    // Poll
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(10 * time.Second):
        }

        status, err := a.pollSemaphoreTask(ctx, taskID)
        if err != nil {
            return err // retryable
        }
        slog.Info("semaphore task poll", "task_name", taskName, "task_id", taskID, "status", status)

        switch status {
        case "finished":
            return nil
        case "failed":
            return temporal.NewNonRetryableApplicationError("seed task failed", "SemaphoreFailed", nil)
        }
    }
}
```

Extract `triggerSemaphoreTask` and `pollSemaphoreTask` as private helpers to keep the public method clean and testable.

### `types.ReleaseInput` — ServiceName Field

Check whether `types.ReleaseInput` already has a `ServiceName string` field (it should be defined in Story 1.1). If not, add it. `ServiceName` is the Kubernetes service/app name (e.g., `fourdogs-central`).

### Error Handling

- Non-2xx from Semaphore → `fmt.Errorf("semaphore: unexpected status %d for %s", resp.StatusCode, url)` — Temporal retries by default
- `"failed"` status → `temporal.NewNonRetryableApplicationError(...)` — Temporal does NOT retry
- Context cancellation (`ctx.Done()`) → return `ctx.Err()` — Temporal handles as cancellation

### Testing with httptest

```go
func TestSeedSecrets_Finished(t *testing.T) {
    var pollCount int
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == http.MethodPost {
            w.WriteHeader(http.StatusAccepted)
            fmt.Fprintln(w, `{"id": 1}`)
            return
        }
        pollCount++
        w.WriteHeader(http.StatusOK)
        fmt.Fprintln(w, `{"id": 1, "status": "finished"}`)
    }))
    defer srv.Close()

    a := &activities.ReleaseActivities{
        SemaphoreBaseURL:   srv.URL,
        SemaphoreAPIToken:  "test",
        SemaphoreProjectID: "proj-1",
    }
    err := a.SeedSecrets(context.Background(), types.ReleaseInput{ServiceName: "fourdogs-central"})
    if err != nil {
        t.Fatalf("want nil, got %v", err)
    }
}
```

Mirror this for `"failed"` path — assert the returned error is NonRetryable.

### References

- [Architecture: SeedSecrets Activity](docs/terminus/platform/releaseorchestrator/architecture.md#seedsecrets)
- [Epics: Story 2.3](docs/terminus/platform/releaseorchestrator/epics.md#story-23-seedsecrets-activity-implementation)
- Temporal: `go.temporal.io/sdk/temporal.NewNonRetryableApplicationError`

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
