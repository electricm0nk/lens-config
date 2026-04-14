# Story 2.5: WaitForArgoCDSync Activity Implementation

Status: backlog

## Story

As a platform developer,
I want the `WaitForArgoCDSync` activity implemented to poll the ArgoCD API until the target application is both `Healthy` and `Synced` so that downstream smoke tests only run against a fully converged deployment,
so that false-positive smoke test failures caused by in-progress ArgoCD reconciliation are eliminated.

## Acceptance Criteria

1. `WaitForArgoCDSync` method on `*ReleaseActivities` polls `GET {ArgoCDBaseURL}/api/v1/applications/{service}` every 15 seconds
2. `activity.RecordHeartbeat(ctx, map[string]string{"health": health, "sync": sync})` is called on EVERY poll cycle before sleep — not just on change
3. When `health == "Healthy" && sync == "Synced"` → return `nil`
4. When context deadline is exceeded → return `temporal.NewNonRetryableApplicationError("argocd sync timeout", "SyncTimeout", nil)`
5. When ArgoCD returns non-200 → return retryable `fmt.Errorf` (Temporal retries)
6. HTTP requests carry `Authorization: Bearer {ArgoCDAPIToken}` header
7. `slog.Info` called with structured key/value on each poll cycle
8. `os.Getenv` is NOT called inside the method
9. `go test ./...` passes — at minimum one test for `Healthy+Synced` path and one for context cancellation path

**Pre-condition:** Story 2.1 complete. Vault path `secret/terminus/default/argocd/api-token` must be seeded by operator (out of scope); ESO must have synced `release-argocd-token` to `temporal` namespace; `ARGOCD_API_TOKEN` and `ARGOCD_BASE_URL` must be set in the worker deployment.

## Tasks / Subtasks

- [ ] Implement `WaitForArgoCDSync` in `internal/activities/release.go` (AC: 1–8)
  - [ ] Poll `GET {ArgoCDBaseURL}/api/v1/applications/{input.ServiceName}` every 15s (AC: 1)
  - [ ] Call `activity.RecordHeartbeat` on EVERY cycle before sleep (AC: 2, MANDATORY)
  - [ ] Parse JSON response for `.status.health.status` and `.status.sync.status` (AC: 3)
  - [ ] Handle context deadline: call `ctx.Err()` check after sleep, return NonRetryable if deadline exceeded (AC: 4)
  - [ ] Handle non-200: retryable error (AC: 5)
  - [ ] `Authorization: Bearer {ArgoCDAPIToken}` header (AC: 6)
  - [ ] `slog.Info` per cycle (AC: 7)
  - [ ] No `os.Getenv` (AC: 8)
- [ ] Add `activity` import from `go.temporal.io/sdk/activity` (needed for `RecordHeartbeat`)
- [ ] Create/update `internal/activities/release_test.go` (AC: 9)
  - [ ] Test: server returns `Healthy+Synced` on first poll → nil
  - [ ] Test: context cancelled after first poll → NonRetryableApplicationError

### Dev Notes

### ArgoCD API Response Shape

```json
{
  "status": {
    "health": {"status": "Healthy"},
    "sync": {"status": "Synced"}
  }
}
```

Define a minimal local struct for unmarshalling — do NOT depend on ArgoCD client library:

```go
type argoCDAppStatus struct {
    Status struct {
        Health struct{ Status string `json:"status"` } `json:"health"`
        Sync   struct{ Status string `json:"status"` } `json:"sync"`
    } `json:"status"`
}
```

Keep this struct unexported and file-scoped in `internal/activities/release.go`.

### RecordHeartbeat — MANDATORY

`activity.RecordHeartbeat` MUST be called on every poll cycle. This is what allows Temporal to detect a stuck activity and kill it when the `StartToCloseTimeout` of 10 minutes is reached. Without heartbeating, the worker can crash and the activity will not be retried until the timeout fires.

```go
activity.RecordHeartbeat(ctx, map[string]string{
    "health": appStatus.Status.Health.Status,
    "sync":   appStatus.Status.Sync.Status,
})
```

Call this BEFORE the `time.After` sleep on each cycle.

### Implementation Pattern

```go
func (a *ReleaseActivities) WaitForArgoCDSync(ctx context.Context, input types.ReleaseInput) error {
    url := fmt.Sprintf("%s/api/v1/applications/%s", a.ArgoCDBaseURL, input.ServiceName)

    for {
        resp, err := a.doGetWithAuth(ctx, url, a.ArgoCDAPIToken)
        if err != nil {
            return err // retryable
        }
        defer resp.Body.Close()

        if resp.StatusCode != http.StatusOK {
            return fmt.Errorf("argocd: unexpected status %d for %s", resp.StatusCode, url)
        }

        var appStatus argoCDAppStatus
        if err := json.NewDecoder(resp.Body).Decode(&appStatus); err != nil {
            return fmt.Errorf("argocd: decode error: %w", err)
        }

        health := appStatus.Status.Health.Status
        sync := appStatus.Status.Sync.Status

        activity.RecordHeartbeat(ctx, map[string]string{"health": health, "sync": sync})
        slog.Info("argocd sync poll", "service", input.ServiceName, "health", health, "sync", sync)

        if health == "Healthy" && sync == "Synced" {
            return nil
        }

        select {
        case <-ctx.Done():
            return temporal.NewNonRetryableApplicationError("argocd sync timeout", "SyncTimeout", nil)
        case <-time.After(15 * time.Second):
        }
    }
}
```

Note: `doGetWithAuth` is a private helper that builds an HTTP GET with `Authorization: Bearer {token}` — extract this for reuse across all activities.

### Time Sensitivity

The `StartToCloseTimeout` for this activity is `10 * time.Minute` (set in Story 2.2). The heartbeat cadence (15s) is well within any reasonable Temporal heartbeat timeout. Do NOT set a separate `HeartbeatTimeout` in the workflow — the `StartToCloseTimeout` is the limiting constraint.

### Test: Context Cancellation

```go
func TestWaitForArgoCDSync_Timeout(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintln(w, `{"status":{"health":{"status":"Progressing"},"sync":{"status":"OutOfSync"}}}`)
    }))
    defer srv.Close()

    a := &activities.ReleaseActivities{
        ArgoCDBaseURL:  srv.URL,
        ArgoCDAPIToken: "test",
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Millisecond)
    defer cancel()

    err := a.WaitForArgoCDSync(ctx, types.ReleaseInput{ServiceName: "fourdogs-central"})
    if err == nil {
        t.Fatal("want error, got nil")
    }
    // Should be NonRetryable
    var appErr *temporal.ApplicationError
    if !errors.As(err, &appErr) || appErr.NonRetryable() == false {
        t.Errorf("want NonRetryableApplicationError, got %T: %v", err, err)
    }
}
```

Note: In tests, bypass the 15s sleep by using a very short context timeout to trigger the `ctx.Done()` branch quickly.

### References

- [Architecture: WaitForArgoCDSync Activity](docs/terminus/platform/releaseorchestrator/architecture.md#waitforargocdSync)
- [Epics: Story 2.5](docs/terminus/platform/releaseorchestrator/epics.md#story-25-wait-for-argocd-sync-activity-implementation)
- Temporal: `go.temporal.io/sdk/activity.RecordHeartbeat`
- ArgoCD REST API: `/api/v1/applications/{name}`

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
