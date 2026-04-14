---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/platform/releaseorchestrator/architecture.md
  - docs/terminus/platform/releaseorchestrator/adversarial-review-report.md
workflowType: epics-and-stories
project_name: terminus-platform-releaseorchestrator
user_name: electricm0nk
date: '2026-04-10'
---

# terminus-platform-releaseorchestrator — Epic Breakdown

## Overview

Complete epic and story breakdown for `terminus-platform-releaseorchestrator`, decomposing architecture requirements into implementable stories. Covers the Go migration of the Temporal worker and implementation of `ReleaseWorkflow`.

## Requirements Inventory

### Functional Requirements

FR1: The system must implement `ReleaseWorkflow` — a Temporal workflow that sequences: SeedSecrets → [ProvisionDatabase] → WaitForArgoCDSync → RunSmokeTest
FR2: The system must implement `SeedSecrets` activity — invoke Semaphore REST API to trigger Ansible seed playbook; fire-and-poll until terminal state
FR3: The system must implement `ProvisionDatabase` activity — invoke Semaphore REST API to trigger Ansible DB provision playbook; fire-and-poll until terminal state; skip entirely when `ProvisionDB: false`
FR4: The system must implement `WaitForArgoCDSync` activity — poll ArgoCD REST API every 15 seconds with `activity.RecordHeartbeat`; succeed when `health.status == Healthy && sync.status == Synced`; fail after 10 minutes
FR5: The system must implement `RunSmokeTest` activity — invoke Semaphore REST API to trigger Ansible verify playbook; fire-and-poll until terminal state
FR6: The system must accept `ReleaseInput` on the `terminus-platform` task queue: `{Service string, SHA string, Env string, ProvisionDB bool}`
FR7: The system must enforce workflow ID pattern `release-{service}-{sha[:8]}` with `ALLOW_DUPLICATE_FAILED_ONLY` reuse policy
FR8: The system must migrate `DailyBriefingWorkflow` TypeScript stub → Go stub (same no-op behavior; required for binary to compile and worker to start)
FR9: The system must replace the Node.js Dockerfile with a Go multi-stage Dockerfile (`gcr.io/distroless/static` runtime)
FR10: The system must remove all TypeScript scaffold files after the Go binary compiles and CI is updated
FR11: The system must declare `ReleaseInput`, `ReleaseResult`, and `ModulePayload[S any]` Go types in `internal/types/`
FR12: The system must provision two `ExternalSecret` k8s manifests: `release-semaphore-token`, `release-argocd-token` targeting Vault paths `secret/terminus/default/semaphore/api-token` and `secret/terminus/default/argocd/api-token`

### Non-Functional Requirements

NFR1: All workflow function code must be deterministic — no `time.Now()`, `os.Getenv`, or `http` calls inside workflow functions; Go SDK panics on violations
NFR2: All credentials delivered via Vault KV → ESO → k8s Secret → env var; never in workflow input (stored in Temporal history)
NFR3: All activities must log with `slog.Info`/`slog.Error` with structured key/value pairs; no `fmt.Println`
NFR4: `WaitForArgoCDSync` must call `activity.RecordHeartbeat(ctx, progress)` on every poll cycle
NFR5: 80% line coverage minimum; 100% on type validation (`ReleaseInput`, `ReleaseResult`)
NFR6: TDD: `_test.go` files scaffolded before implementation for all new code
NFR7: All `os.Getenv` calls belong in `cmd/worker/main.go` or activity constructors — not deep in activity logic
NFR8: No `interface{}` in workflow or activity function signatures — typed structs only
NFR9: Unrecoverable errors returned as `temporal.NewNonRetryableApplicationError(...)`
NFR10: Go module name: `github.com/electricm0nk/terminus-platform`; Go version: 1.25

### Additional Requirements

- **Migration atomicity constraint**: Go binary must compile and CI must reference the new Dockerfile **before** TypeScript scaffold is removed (adversarial review Finding 3 — story ordering constraint)
- **Vault seeding prerequisite**: All activity stories consuming `SEMAPHORE_API_TOKEN` or `ARGOCD_API_TOKEN` must include explicit AC pre-condition: _"Vault paths seeded and ESO k8s Secrets exist"_ (adversarial review Finding 2)
- **No starter template**: Migration of existing repo; standard Go module layout, not greenfield
- **No new Helm chart**: Same `temporal-worker` chart; `values.yaml` unchanged
- **Worker registration**: `cmd/worker/main.go` must register both `DailyBriefingWorkflow` + `ReleaseWorkflow` and their respective activities

### FR Coverage Map

FR1: Epic 2 — ReleaseWorkflow sequences activities
FR2: Epic 2 — SeedSecrets activity
FR3: Epic 2 — ProvisionDatabase activity (conditional)
FR4: Epic 2 — WaitForArgoCDSync activity
FR5: Epic 2 — RunSmokeTest activity
FR6: Epic 2 — ReleaseInput type + task queue registration
FR7: Epic 2 — Workflow ID policy in main.go / workflow options
FR8: Epic 1 — DailyBriefingWorkflow Go stub (migration prerequisite)
FR9: Epic 1 — Go Dockerfile (migration prerequisite, atomicity constraint)
FR10: Epic 1 — TypeScript removal (last story in migration epic)
FR11: Epic 1 — Go types foundation (`internal/types/`)
FR12: Epic 3 — k8s ExternalSecret manifests for credential delivery

## Epic List

### Epic 1: Go Worker Foundation
Establish the Go module, compile a working binary with all required stubs, update the CI/CD pipeline to the Go Dockerfile, and remove the TypeScript scaffold — leaving a clean, deployable Go Temporal worker as the base for feature work.
**FRs covered:** FR8, FR9, FR10, FR11

### Epic 2: Release Workflow
Implement `ReleaseWorkflow` and all four release activities (`SeedSecrets`, `ProvisionDatabase`, `WaitForArgoCDSync`, `RunSmokeTest`) — enabling automated, orchestrated service deployments triggered from Semaphore CI.
**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR6, FR7

### Epic 3: Credential Infrastructure
Provision the k8s `ExternalSecret` manifests for Semaphore and ArgoCD API tokens, enabling the release activities to authenticate with external systems via the Vault → ESO → k8s Secret chain.
**FRs covered:** FR12

---

## Epic 1: Go Worker Foundation

Establish the Go module, compile a working binary with all required stubs, update the CI/CD pipeline to the Go Dockerfile, and remove the TypeScript scaffold — leaving a clean, deployable Go Temporal worker as the base for feature work.

**Goal:** A deployable Go Temporal worker binary that registers `DailyBriefingWorkflow` (stub) and connects to the `terminus-platform` task queue, replacing the Node.js worker entirely.

---

### Story 1.1: Go Module Scaffold and Type Foundation

As a platform developer,
I want a Go module initialised at `github.com/electricm0nk/terminus-platform` with `internal/types/` declaring `ModulePayload[S any]`, `RunContext`, `ReleaseInput`, and `ReleaseResult`,
So that all subsequent stories have a typed foundation to build on with no `interface{}` in signatures.

**Acceptance Criteria:**

**Given** the repository currently contains a TypeScript workspace (`package.json`, `tsconfig.json`, `services/`, `shared/`)
**When** the Go module scaffold is created
**Then** `go.mod` exists at repo root declaring `module github.com/electricm0nk/terminus-platform` and `go 1.25`
**And** `go.sum` is present (may be empty until dependencies are added in story 1.2)
**And** `internal/types/module.go` declares `RunContext`, `ModulePayload[S any]`, and `Module` interface matching the TypeScript originals semantically
**And** `internal/types/release.go` declares `ReleaseInput{Service, SHA, Env string; ProvisionDB bool}` and `ReleaseResult{Status, CompletedAt string; SmokeTestPassed bool}`
**And** `internal/types/module_test.go` and `internal/types/release_test.go` exist with 100% coverage on type construction and zero-value behaviour
**And** `go test ./internal/types/...` passes
**And** the TypeScript files are NOT yet removed (TypeScript scaffold coexists; removal is Story 1.4)

**References:** FR11, NFR6, NFR8, NFR10

---

### Story 1.2: DailyBriefing Go Stub and Worker Entrypoint

As a platform developer,
I want the `DailyBriefingWorkflow` Go stub and `cmd/worker/main.go` entrypoint wired together so that the worker binary compiles and registers on the `terminus-platform` task queue,
So that the Go worker is a valid replacement for the Node.js worker at the binary level.

**Acceptance Criteria:**

**Given** Story 1.1 types are in place
**When** the Go workflow stub and worker entrypoint are implemented
**Then** `internal/workflows/daily_briefing.go` declares `DailyBriefingWorkflow` accepting `types.ModulePayload[DailyBriefingSchema]` and performing the same payload validation (date + timezone required) as the TypeScript original — no other logic
**And** `internal/workflows/daily_briefing_test.go` covers: valid payload passes, missing date returns error, missing timezone returns error
**And** `internal/activities/briefing.go` declares `FetchBriefingData` activity returning a stub `ModulePayload` with empty sources — matching TypeScript semantics
**And** `internal/activities/briefing_test.go` covers: happy path returns non-nil result; error path wraps as `temporal.NewNonRetryableApplicationError`
**And** `cmd/worker/main.go` reads `TEMPORAL_ADDRESS`, `TEMPORAL_NAMESPACE`, `TEMPORAL_TASK_QUEUE` from `os.Getenv` (with same defaults as TypeScript original)
**And** `cmd/worker/main.go` registers `DailyBriefingWorkflow` and `briefingActivities` on the worker
**And** `go build ./cmd/worker/` produces a binary without errors
**And** `go test ./...` passes with ≥ 80% line coverage
**And** no `time.Now()`, `os.Getenv`, or `http` calls exist inside `internal/workflows/` files
**And** all activity logging uses `slog.Info`/`slog.Error` with structured key/value pairs

**References:** FR8, NFR1, NFR3, NFR6, NFR7, NFR9

---

### Story 1.3: Go Dockerfile and CI Pipeline Update

As a platform developer,
I want a Go multi-stage Dockerfile and updated CI pipeline so that the worker image builds from Go source and is pushed to GHCR under the same image name,
So that the existing Helm chart and ArgoCD deployment work without modification.

**Acceptance Criteria:**

**Given** `go build ./cmd/worker/` succeeds (Story 1.2 complete)
**When** the Dockerfile and CI are updated
**Then** `Dockerfile` at repo root is a multi-stage Go build: stage 1 uses `golang:1.25` to `go build -o /worker ./cmd/worker/`, stage 2 copies binary into `gcr.io/distroless/static:nonroot`
**And** the final image runs as non-root (distroless nonroot image enforces this)
**And** the image is still tagged and pushed to `ghcr.io/electricm0nk/terminus-platform-worker` (same registry path as TypeScript image)
**And** `helm/temporal-worker/values.yaml` is unchanged — `image.repository` and `image.tag` are the same
**And** the CI pipeline Dockerfile path reference is updated to point to the new `Dockerfile` at repo root (was `services/temporal/Dockerfile`)
**And** `docker build .` completes locally without errors
**And** the built image entrypoint is the Go binary (not `node services/temporal/dist/worker.js`)

**Pre-condition:** Story 1.2 is complete and `go build ./cmd/worker/` passes.

**References:** FR9, NFR2

---

### Story 1.4: TypeScript Scaffold Removal

As a platform developer,
I want all TypeScript-era files removed from the repository,
So that the codebase is a clean Go module with no npm artefacts, no `node_modules`, and no ambiguity about the runtime.

**Acceptance Criteria:**

**Given** Story 1.3 is complete — Go binary compiles, Dockerfile builds, CI pipeline points to new Dockerfile
**When** the TypeScript scaffold is removed
**Then** the following are deleted from the repository: `services/` directory, `shared/` directory, `node_modules/` directory, `coverage/` directory, `package.json`, `package-lock.json`, `tsconfig.json`
**And** `go build ./cmd/worker/` still passes after removal
**And** `go test ./...` still passes after removal
**And** `git status` shows only Go files and Helm/k8s/docs files in the repository
**And** no `node`, `npm`, or TypeScript references remain in any non-docs file

**Pre-condition:** Story 1.3 is complete. This story MUST NOT be merged before Story 1.3 is merged. Story ordering constraint from adversarial review Finding 3.

**References:** FR10

---

## Epic 2: Release Workflow

Implement `ReleaseWorkflow` and all four release activities, enabling automated orchestrated service deployments triggered from Semaphore CI.

**Goal:** A fully working `ReleaseWorkflow` that can be triggered via `temporal workflow start` CLI, sequences all deployment activities, and returns a `ReleaseResult` with completion status.

---

### Story 2.1: Release Activity Stubs

As a platform developer,
I want all four release activity function signatures declared with stub implementations and full test scaffolding,
So that the workflow can reference typed activity functions from Story 2.2 onwards without compilation errors.

**Acceptance Criteria:**

**Given** Epic 1 is complete (Go worker binary builds and deploys)
**When** release activity stubs are added
**Then** `internal/activities/release.go` declares four exported functions: `SeedSecrets(ctx context.Context, input ReleaseInput) error`, `ProvisionDatabase(ctx context.Context, input ReleaseInput) error`, `WaitForArgoCDSync(ctx context.Context, input ReleaseInput) error`, `RunSmokeTest(ctx context.Context, input ReleaseInput) error`
**And** all four stub implementations return `nil` (no-op)
**And** `internal/activities/release_test.go` exists with test scaffolding for all four functions (happy path only at this stage — stubs return nil)
**And** `go build ./...` passes
**And** `go test ./...` passes
**And** no `interface{}` appears in any release activity signature

**References:** FR2, FR3, FR4, FR5, NFR8

---

### Story 2.2: ReleaseWorkflow Orchestration

As a platform developer,
I want `ReleaseWorkflow` implemented as a pure deterministic Temporal workflow that sequences the four release activities with correct conditional logic and retry policy,
So that the workflow can be triggered and will execute the correct activities in the correct order.

**Acceptance Criteria:**

**Given** Story 2.1 stubs are in place
**When** `ReleaseWorkflow` is implemented
**Then** `internal/workflows/release.go` declares `ReleaseWorkflow(ctx workflow.Context, input types.ReleaseInput) (types.ReleaseResult, error)`
**And** the workflow calls `SeedSecrets` first on all runs
**And** the workflow calls `ProvisionDatabase` only when `input.ProvisionDB == true`; when `false` the activity is not called at all (not a no-op call)
**And** the workflow calls `WaitForArgoCDSync` after secrets/provisioning steps
**And** the workflow calls `RunSmokeTest` last
**And** workflow activity options specify `StartToCloseTimeout: 10 * time.Minute` for `WaitForArgoCDSync`; all others use default retry policy
**And** `internal/workflows/release_test.go` covers: full run with `ProvisionDB: true`, full run with `ProvisionDB: false` (ProvisionDatabase not called), activity failure propagates as workflow failure
**And** no `time.Now()`, `os.Getenv`, or `http` calls exist in `internal/workflows/release.go`
**And** `go test ./internal/workflows/...` passes
**And** `cmd/worker/main.go` registers `ReleaseWorkflow` and `releaseActivities`

**References:** FR1, FR6, FR7, NFR1, NFR6

---

### Story 2.3: SeedSecrets Activity Implementation

As a platform developer,
I want `SeedSecrets` implemented to invoke a Semaphore task via REST API and poll for completion,
So that Vault secrets are seeded as the first step of every release.

**Pre-condition:** Vault paths `secret/terminus/default/semaphore/api-token` are seeded and ESO has synced `release-semaphore-token` k8s Secret. `SEMAPHORE_API_TOKEN` and `SEMAPHORE_PROJECT_ID` env vars are set on the worker deployment.

**Acceptance Criteria:**

**Given** the pre-condition is met
**When** `SeedSecrets` is called with a valid `ReleaseInput`
**Then** the activity reads `SEMAPHORE_API_TOKEN` and `SEMAPHORE_PROJECT_ID` from the environment (via constructor injection into the activity — not `os.Getenv` inside the function body)
**And** the activity issues `POST /api/v1alpha1/project/{id}/tasks` to the Semaphore API with task name derived from `input.Service` (e.g. `seed-fourdogs-central-secrets`)
**And** the activity polls `GET /api/v1alpha1/project/{id}/tasks/{task_id}` every 10 seconds until status is `"finished"` or `"failed"`
**And** if status is `"finished"` the activity returns nil
**And** if status is `"failed"` the activity returns `temporal.NewNonRetryableApplicationError("seed task failed", "SemaphoreFailed", nil)`
**And** the activity logs `slog.Info("seed-secrets started", "service", input.Service)` on entry and `slog.Info("seed-secrets complete", "service", input.Service)` on success
**And** `internal/activities/release_test.go` covers: Semaphore returns `"finished"` → activity returns nil; Semaphore returns `"failed"` → activity returns NonRetryableApplicationError; HTTP error on poll → error propagated
**And** `go test ./internal/activities/...` passes

**References:** FR2, NFR2, NFR3, NFR7, NFR9

---

### Story 2.4: ProvisionDatabase Activity Implementation

As a platform developer,
I want `ProvisionDatabase` implemented to invoke a Semaphore task for DB provisioning, using the same fire-and-poll pattern as `SeedSecrets`,
So that databases are provisioned as part of first-deploy releases when `ProvisionDB: true`.

**Pre-condition:** Same as Story 2.3 (SEMAPHORE_API_TOKEN, SEMAPHORE_PROJECT_ID set).

**Acceptance Criteria:**

**Given** the pre-condition is met
**When** `ProvisionDatabase` is called with `input.ProvisionDB == true`
**Then** the activity uses the same Semaphore fire-and-poll pattern as `SeedSecrets` with task name `db-provision-{service}` (e.g. `db-provision-fourdogs-central`)
**And** on Semaphore task `"finished"` returns nil
**And** on Semaphore task `"failed"` returns `temporal.NewNonRetryableApplicationError("db provision failed", "SemaphoreFailed", nil)`
**And** activity logs entry and completion with `slog.Info`
**And** note: the workflow (Story 2.2) is responsible for skipping this activity when `ProvisionDB: false` — this activity does not need to check the flag
**And** `internal/activities/release_test.go` is updated with the same three test cases as Story 2.3
**And** `go test ./internal/activities/...` passes

**References:** FR3, NFR2, NFR3, NFR7, NFR9

---

### Story 2.5: WaitForArgoCDSync Activity Implementation

As a platform developer,
I want `WaitForArgoCDSync` implemented to poll the ArgoCD REST API with a Temporal heartbeat on every cycle,
So that the workflow waits for ArgoCD to successfully sync the new image before proceeding to smoke testing.

**Pre-condition:** Vault path `secret/terminus/default/argocd/api-token` is seeded and ESO has synced `release-argocd-token` k8s Secret. `ARGOCD_API_TOKEN` and `ARGOCD_BASE_URL` env vars are set on the worker deployment.

**Acceptance Criteria:**

**Given** the pre-condition is met
**When** `WaitForArgoCDSync` is called with a valid `ReleaseInput`
**Then** the activity reads `ARGOCD_API_TOKEN` and `ARGOCD_BASE_URL` from the environment (via constructor injection)
**And** the activity polls `GET {ARGOCD_BASE_URL}/api/v1/applications/{service}` every 15 seconds
**And** on every poll cycle (before sleeping) the activity calls `activity.RecordHeartbeat(ctx, map[string]string{"health": healthStatus, "sync": syncStatus})`
**And** when `health.status == "Healthy"` AND `sync.status == "Synced"` the activity returns nil
**And** if the context deadline is exceeded (10-minute `StartToCloseTimeout` set by workflow) the activity returns `temporal.NewNonRetryableApplicationError("argocd sync timeout", "SyncTimeout", nil)`
**And** if the ArgoCD API returns a non-200 response the activity returns a retryable error (plain `fmt.Errorf`)
**And** activity logs `slog.Info("argocd-sync poll", "health", healthStatus, "sync", syncStatus)` on each cycle
**And** `internal/activities/release_test.go` is updated to cover: immediate Healthy+Synced → returns nil; requires 2 polls then Healthy+Synced → returns nil; context cancelled → returns error with RecordHeartbeat called
**And** `go test ./internal/activities/...` passes

**References:** FR4, NFR2, NFR3, NFR4, NFR7, NFR9

---

### Story 2.6: RunSmokeTest Activity Implementation

As a platform developer,
I want `RunSmokeTest` implemented to invoke a Semaphore verification task via the same fire-and-poll pattern,
So that every release is validated by an automated smoke test before the workflow completes.

**Pre-condition:** Same as Story 2.3 (SEMAPHORE_API_TOKEN, SEMAPHORE_PROJECT_ID set).

**Acceptance Criteria:**

**Given** the pre-condition is met
**When** `RunSmokeTest` is called with a valid `ReleaseInput`
**Then** the activity uses the same Semaphore fire-and-poll pattern with task name `verify-{service}-deploy` (e.g. `verify-fourdogs-central-deploy`)
**And** on Semaphore task `"finished"` the activity returns nil
**And** on Semaphore task `"failed"` returns `temporal.NewNonRetryableApplicationError("smoke test failed", "SmokeTestFailed", nil)`
**And** activity logs entry and completion with `slog.Info`
**And** `internal/activities/release_test.go` is updated with same three test cases as Story 2.3
**And** `go test ./...` passes with ≥ 80% line coverage across the module

**References:** FR5, NFR2, NFR3, NFR7, NFR9

---

## Epic 3: Credential Infrastructure

Provision the k8s `ExternalSecret` manifests for Semaphore and ArgoCD API tokens, enabling the release activities to authenticate with external systems via the Vault → ESO → k8s Secret chain.

**Goal:** Two `ExternalSecret` resources deployed in the `temporal` namespace that sync Semaphore and ArgoCD API tokens from Vault into k8s Secrets mounted as env vars on the worker Deployment.

---

### Story 3.1: ExternalSecret Manifests for Release Credentials

As a platform operator,
I want two `ExternalSecret` k8s manifests committed to the repository so that ESO automatically syncs Semaphore and ArgoCD API tokens from Vault into the temporal namespace,
So that release activities can authenticate with Semaphore and ArgoCD without credentials being stored in code or CI.

**Pre-condition:** Vault paths `secret/terminus/default/semaphore/api-token` and `secret/terminus/default/argocd/api-token` are seeded by the operator (out of scope for this story — operator task).

**Acceptance Criteria:**

**Given** a Vault instance with the two API token paths seeded
**When** the ExternalSecret manifests are applied to the cluster
**Then** `k8s/external-secret-release-semaphore.yaml` defines an `ExternalSecret` in namespace `temporal` named `release-semaphore-token` referencing Vault path `secret/terminus/default/semaphore/api-token` and projecting to k8s Secret key `SEMAPHORE_API_TOKEN`
**And** `k8s/external-secret-release-argocd.yaml` defines an `ExternalSecret` in namespace `temporal` named `release-argocd-token` referencing Vault path `secret/terminus/default/argocd/api-token` and projecting to k8s Secret key `ARGOCD_API_TOKEN`
**And** both manifests follow the same `SecretStore` reference and `refreshInterval` pattern as existing ESO manifests in the repository
**And** `kubectl apply -f k8s/external-secret-release-semaphore.yaml -f k8s/external-secret-release-argocd.yaml` applies without errors against the dev cluster
**And** after apply, `kubectl get secret release-semaphore-token -n temporal` exists and contains `SEMAPHORE_API_TOKEN`
**And** after apply, `kubectl get secret release-argocd-token -n temporal` exists and contains `ARGOCD_API_TOKEN`

**References:** FR12, NFR2

---

## Validation Summary

### FR Coverage

| FR | Story | Status |
|---|---|---|
| FR1 | Story 2.2 | ✅ |
| FR2 | Story 2.3 | ✅ |
| FR3 | Story 2.4 | ✅ |
| FR4 | Story 2.5 | ✅ |
| FR5 | Story 2.6 | ✅ |
| FR6 | Story 2.1, 2.2 | ✅ |
| FR7 | Story 2.2 | ✅ |
| FR8 | Story 1.2 | ✅ |
| FR9 | Story 1.3 | ✅ |
| FR10 | Story 1.4 | ✅ |
| FR11 | Story 1.1 | ✅ |
| FR12 | Story 3.1 | ✅ |

All 12 FRs covered. No gaps.

### NFR Coverage

| NFR | Stories |
|---|---|
| NFR1 (determinism) | 2.2 AC |
| NFR2 (credentials via ESO) | 1.3, 2.3, 2.4, 2.5, 2.6, 3.1 pre-conditions + AC |
| NFR3 (slog logging) | 2.3, 2.4, 2.5, 2.6 AC |
| NFR4 (RecordHeartbeat) | 2.5 AC |
| NFR5 (80% coverage) | 1.2, 2.6 AC |
| NFR6 (TDD) | All stories require `_test.go` before implementation |
| NFR7 (os.Getenv in constructor) | 2.3, 2.4, 2.5, 2.6 AC |
| NFR8 (typed signatures) | 2.1 AC |
| NFR9 (NonRetryableApplicationError) | 2.3, 2.4, 2.5, 2.6 AC |
| NFR10 (Go 1.25, module name) | 1.1 AC |

### Adversarial Review Constraint Compliance

- **Finding 2 (Vault seeding pre-condition):** Explicit pre-condition in Stories 2.3, 2.4, 2.5, 2.6, 3.1 ✅
- **Finding 3 (Migration atomicity):** Story 1.4 has explicit pre-condition blocking on Story 1.3 merge ✅

### Dependency Order

```
1.1 → 1.2 → 1.3 → 1.4   (Epic 1: must be sequential)
Epic 1 complete → 2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 2.6
Epic 3: Story 3.1 is independent; can run in parallel with Epic 2
```
