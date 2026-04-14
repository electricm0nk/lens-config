# Story 2.2: ReleaseWorkflow Orchestration

Status: backlog

## Story

As a platform developer,
I want `ReleaseWorkflow` implemented in `internal/workflows/release.go` orchestrating the four activities in sequence so that a single Temporal workflow execution drives the full deployment pipeline,
so that operators can trigger a release by starting one workflow and observe it to completion.

## Acceptance Criteria

1. `internal/workflows/release.go` is created with function `ReleaseWorkflow(ctx workflow.Context, input types.ReleaseInput) (types.ReleaseResult, error)`
2. Activity execution order: `SeedSecrets` → `ProvisionDatabase` (if `input.ProvisionDB == true`) → `WaitForArgoCDSync` → `RunSmokeTest`
3. `WaitForArgoCDSync` is called with `ActivityOptions{StartToCloseTimeout: 10 * time.Minute}`; all other activities use default `ActivityOptions`
4. On `RunSmokeTest` returning `nil`, the workflow returns `types.ReleaseResult{SmokeTestPassed: true}`
5. On any activity returning a non-nil error, the workflow returns that error immediately (no custom retry logic in the workflow — activities own their retry policy)
6. `ReleaseWorkflow` is registered in `cmd/worker/main.go` with `w.RegisterWorkflow(workflows.ReleaseWorkflow)`
7. No `time.Now()`, `rand`, `uuid`, or other non-deterministic calls inside the workflow function
8. `go build ./...` passes
9. `go test ./...` passes — at minimum one test covering the ProvisionDB conditional branch

**Pre-condition:** Story 2.1 is complete (`ReleaseActivities` struct and stubs exist).

## Tasks / Subtasks

- [ ] Create `internal/workflows/release.go` (AC: 1–5, 7)
  - [ ] `ReleaseWorkflow` signature: `(ctx workflow.Context, input types.ReleaseInput) (types.ReleaseResult, error)`
  - [ ] Set `ActivityOptions` with `StartToCloseTimeout: 10 * time.Minute` for `WaitForArgoCDSync`; use `workflow.WithActivityOptions(ctx, ao)` pattern
  - [ ] Use separate `activityCtx` for `WaitForArgoCDSync` to avoid affecting other activities
  - [ ] `if input.ProvisionDB` guard around `ProvisionDatabase` call (AC: 2)
  - [ ] Return `types.ReleaseResult{SmokeTestPassed: true}` on `RunSmokeTest` nil return (AC: 4)
  - [ ] No non-deterministic calls (AC: 7)
- [ ] Register `ReleaseWorkflow` in `cmd/worker/main.go` (AC: 6)
- [ ] Create `internal/workflows/release_test.go` (AC: 9)
  - [ ] Test with `ProvisionDB: false` — verify `ProvisionDatabase` is NOT called
  - [ ] Test with `ProvisionDB: true` — verify `ProvisionDatabase` IS called
  - [ ] Use Temporal's `testsuite` package: `testsuite.WorkflowTestSuite` and `env.ExecuteWorkflow`
- [ ] Run `go build ./...` and `go test ./...` (AC: 8, 9)

## Dev Notes

### ReleaseWorkflow Implementation

Create `internal/workflows/release.go`:

```go
package workflows

import (
	"time"

	"go.temporal.io/sdk/workflow"

	"github.com/electricm0nk/terminus-platform/internal/activities"
	"github.com/electricm0nk/terminus-platform/internal/types"
)

func ReleaseWorkflow(ctx workflow.Context, input types.ReleaseInput) (types.ReleaseResult, error) {
	ao := workflow.ActivityOptions{
		StartToCloseTimeout: 5 * time.Second, // default — activities own retry policy
	}
	ctx = workflow.WithActivityOptions(ctx, ao)

	// SeedSecrets
	var a *activities.ReleaseActivities
	if err := workflow.ExecuteActivity(ctx, a.SeedSecrets, input).Get(ctx, nil); err != nil {
		return types.ReleaseResult{}, err
	}

	// ProvisionDatabase — conditional
	if input.ProvisionDB {
		if err := workflow.ExecuteActivity(ctx, a.ProvisionDatabase, input).Get(ctx, nil); err != nil {
			return types.ReleaseResult{}, err
		}
	}

	// WaitForArgoCDSync — extended timeout
	syncCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		StartToCloseTimeout: 10 * time.Minute,
	})
	if err := workflow.ExecuteActivity(syncCtx, a.WaitForArgoCDSync, input).Get(ctx, nil); err != nil {
		return types.ReleaseResult{}, err
	}

	// RunSmokeTest
	if err := workflow.ExecuteActivity(ctx, a.RunSmokeTest, input).Get(ctx, nil); err != nil {
		return types.ReleaseResult{}, err
	}

	return types.ReleaseResult{SmokeTestPassed: true}, nil
}
```

**Key pattern — `var a *activities.ReleaseActivities`**: The nil pointer is intentional. Temporal uses the method reference for registration lookup — it does NOT call the method on the nil receiver. The actual execution uses the registered `*ReleaseActivities` instance from `main.go`.

### Registration in main.go

Add alongside `DailyBriefingWorkflow`:

```go
w.RegisterWorkflow(workflows.ReleaseWorkflow)
```

### Test File

Use Temporal's test suite. The test suite mocks activity execution via `env.OnActivity`.

```go
package workflows_test

import (
	"testing"

	"go.temporal.io/sdk/testsuite"

	"github.com/electricm0nk/terminus-platform/internal/activities"
	"github.com/electricm0nk/terminus-platform/internal/types"
	"github.com/electricm0nk/terminus-platform/internal/workflows"
)

func TestReleaseWorkflow_WithProvisionDB_False(t *testing.T) {
	suite := testsuite.WorkflowTestSuite{}
	env := suite.NewTestWorkflowEnvironment()

	var a *activities.ReleaseActivities
	env.OnActivity(a.SeedSecrets, mock.Anything, mock.Anything).Return(nil)
	env.OnActivity(a.WaitForArgoCDSync, mock.Anything, mock.Anything).Return(nil)
	env.OnActivity(a.RunSmokeTest, mock.Anything, mock.Anything).Return(nil)
	// ProvisionDatabase should NOT be called when ProvisionDB is false

	env.ExecuteWorkflow(workflows.ReleaseWorkflow, types.ReleaseInput{ProvisionDB: false})
	if !env.IsWorkflowCompleted() {
		t.Fatal("workflow not completed")
	}
	var result types.ReleaseResult
	if err := env.GetWorkflowResult(&result); err != nil {
		t.Fatalf("workflow error: %v", err)
	}
	if !result.SmokeTestPassed {
		t.Error("want SmokeTestPassed=true, got false")
	}
}
```

Duplicate for `ProvisionDB: true` with `env.OnActivity(a.ProvisionDatabase, ...)` added.

Note: import `github.com/stretchr/testify/mock` for `mock.Anything` — or use Temporal's built-in matchers.

### IR-2 Resolution

This story resolves Implementation Readiness Note IR-2: "Workflow must set `ReleaseResult{SmokeTestPassed: true}` on `RunSmokeTest` nil return." Implemented in the final `return` statement above.

### Determinism Rules

Inside `ReleaseWorkflow`, **never** use:
- `time.Now()` — use `workflow.Now(ctx)` if you need current time
- `rand.*` — use `workflow.GetInfo(ctx).Attempt` or a seeded source passed as input
- `os.*`, direct I/O, goroutines, `go func()` — all forbidden in workflow code
- `http.Get` or any network call — use activities for all I/O

### References

- [Architecture: Workflow Design](docs/terminus/platform/releaseorchestrator/architecture.md#workflow-design)
- [Architecture: Activity Orchestration](docs/terminus/platform/releaseorchestrator/architecture.md)
- [Epics: Story 2.2](docs/terminus/platform/releaseorchestrator/epics.md#story-22-releaseworkflow-orchestration)
- [Implementation Readiness: IR-2](docs/terminus/platform/releaseorchestrator/implementation-readiness.md)
- Temporal Go SDK: `workflow.ExecuteActivity`, `workflow.WithActivityOptions`

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
