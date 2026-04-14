# Story 2.1: Release Activity Stubs

Status: backlog

## Story

As a platform developer,
I want stub implementations of all four release activities compiled and registered in the worker so that `go build ./...` and `go test ./...` pass with the full activity API surface defined,
so that subsequent stories can implement each activity independently without API drift.

## Acceptance Criteria

1. `internal/activities/release.go` is created
2. `ReleaseActivities` struct is defined with fields: `SemaphoreAPIToken`, `SemaphoreProjectID`, `ArgoCDAPIToken`, `ArgoCDBaseURL` (all `string`)
3. Four methods are defined on `*ReleaseActivities`:
   - `SeedSecrets(ctx context.Context, input types.ReleaseInput) error`
   - `ProvisionDatabase(ctx context.Context, input types.ReleaseInput) error`
   - `WaitForArgoCDSync(ctx context.Context, input types.ReleaseInput) error`
   - `RunSmokeTest(ctx context.Context, input types.ReleaseInput) error`
4. All four stubs return `nil`
5. `ReleaseActivities` is registered in `cmd/worker/main.go` using `w.RegisterActivity(&activities.ReleaseActivities{...})`
6. All four env vars are injected in `main.go`: `SEMAPHORE_API_TOKEN`, `SEMAPHORE_PROJECT_ID`, `ARGOCD_API_TOKEN`, `ARGOCD_BASE_URL`
7. `go build ./...` passes
8. `go test ./...` passes (test file with at least one stub-registration test)

**Pre-condition:** Epic 1 is complete (Stories 1.1–1.4 all merged).

## Tasks / Subtasks

- [ ] Create `internal/activities/release.go` (AC: 1–4)
  - [ ] Define `ReleaseActivities` struct with four `string` fields (AC: 2)
  - [ ] Implement four nil-returning stub methods with correct signatures (AC: 3, 4)
  - [ ] Use `context.Context` from `"context"` — not from Temporal
  - [ ] Use `types.ReleaseInput` parameter (import `internal/types`)
- [ ] Update `cmd/worker/main.go` (AC: 5, 6)
  - [ ] Read four env vars with `os.Getenv` in `main.go` (only here — never inside activity methods)
  - [ ] Pass values into `&activities.ReleaseActivities{...}` struct literal
  - [ ] `w.RegisterActivity(&activities.ReleaseActivities{...})`
- [ ] Create `internal/activities/release_test.go` (AC: 8)
  - [ ] Test that struct can be instantiated with known field values
  - [ ] Test that all four methods return `nil` when called with zero-value context and input
- [ ] Run `go build ./...` and `go test ./...` — verify both pass (AC: 7, 8)

## Dev Notes

### ReleaseActivities Struct

Create `internal/activities/release.go`:

```go
package activities

import (
	"context"

	"github.com/electricm0nk/terminus-platform/internal/types"
)

// ReleaseActivities holds credentials injected at worker startup.
// All fields are read from env vars in cmd/worker/main.go — never os.Getenv inside methods.
type ReleaseActivities struct {
	SemaphoreAPIToken  string
	SemaphoreProjectID string
	ArgoCDAPIToken     string
	ArgoCDBaseURL      string
}

func (a *ReleaseActivities) SeedSecrets(ctx context.Context, input types.ReleaseInput) error {
	return nil
}

func (a *ReleaseActivities) ProvisionDatabase(ctx context.Context, input types.ReleaseInput) error {
	return nil
}

func (a *ReleaseActivities) WaitForArgoCDSync(ctx context.Context, input types.ReleaseInput) error {
	return nil
}

func (a *ReleaseActivities) RunSmokeTest(ctx context.Context, input types.ReleaseInput) error {
	return nil
}
```

### Registration in main.go

The existing `cmd/worker/main.go` already registers `DailyBriefingWorkflow` and `BriefingActivities`. Add release registration alongside:

```go
// Release activities
releaseActivities := &activities.ReleaseActivities{
    SemaphoreAPIToken:  os.Getenv("SEMAPHORE_API_TOKEN"),
    SemaphoreProjectID: os.Getenv("SEMAPHORE_PROJECT_ID"),
    ArgoCDAPIToken:     os.Getenv("ARGOCD_API_TOKEN"),
    ArgoCDBaseURL:      os.Getenv("ARGOCD_BASE_URL"),
}
w.RegisterActivity(releaseActivities)
```

### Anti-patterns — Forbidden

- `os.Getenv` inside any activity method — inject via struct only
- `fmt.Println` — use `slog.Info`, `slog.Error`
- Defining `ReleaseActivities` in the workflow package — must be in `internal/activities/`
- Using `interface{}` or `any` in the activity signature

### Test File

Create `internal/activities/release_test.go`:

```go
package activities_test

import (
	"context"
	"testing"

	"github.com/electricm0nk/terminus-platform/internal/activities"
	"github.com/electricm0nk/terminus-platform/internal/types"
)

func TestReleaseActivities_StubsReturnNil(t *testing.T) {
	a := &activities.ReleaseActivities{
		SemaphoreAPIToken:  "test-token",
		SemaphoreProjectID: "test-project",
		ArgoCDAPIToken:     "test-argocd-token",
		ArgoCDBaseURL:      "https://argocd.example.com",
	}
	input := types.ReleaseInput{}
	ctx := context.Background()

	if err := a.SeedSecrets(ctx, input); err != nil {
		t.Errorf("SeedSecrets: want nil, got %v", err)
	}
	if err := a.ProvisionDatabase(ctx, input); err != nil {
		t.Errorf("ProvisionDatabase: want nil, got %v", err)
	}
	if err := a.WaitForArgoCDSync(ctx, input); err != nil {
		t.Errorf("WaitForArgoCDSync: want nil, got %v", err)
	}
	if err := a.RunSmokeTest(ctx, input); err != nil {
		t.Errorf("RunSmokeTest: want nil, got %v", err)
	}
}
```

### IR-3 Resolution

This story resolves Implementation Readiness Note IR-3: "Define `ReleaseActivities` struct in Story 2.1 for constructor injection pattern." The struct is defined here, methods are stubs. Stories 2.3–2.6 implement each method's body.

### References

- [Architecture: Activity Design](docs/terminus/platform/releaseorchestrator/architecture.md#activity-design)
- [Architecture: Constructor Injection Pattern](docs/terminus/platform/releaseorchestrator/architecture.md#constructor-injection-pattern)
- [Epics: Story 2.1](docs/terminus/platform/releaseorchestrator/epics.md#story-21-release-activity-stubs)
- [Implementation Readiness: IR-3](docs/terminus/platform/releaseorchestrator/implementation-readiness.md)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
