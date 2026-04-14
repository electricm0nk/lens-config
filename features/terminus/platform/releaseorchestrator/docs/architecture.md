---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-04-10'
inputDocuments:
  - docs/terminus/architecture.md
  - docs/terminus/platform/temporal/architecture.md
  - docs/terminus/platform/project-context.md
  - docs/fourdogs/central/foundation/Stories/dev-story-1-5-helm-chart-and-argocd-deployment-to-k3s.md
workflowType: architecture
project_name: terminus-platform-releaseorchestrator
user_name: electricm0nk
date: '2026-04-10'
---

# Architecture Decision Document — ReleaseWorkflow on terminus-platform

_Collaborative step-by-step architecture. Built on 2026-04-10._

---

## Project Context (Step 2)

**Initiative:** terminus-platform-releaseorchestrator
**Track:** tech-change
**Target repo:** terminus.platform
**Layer:** terminus-platform (consumer of infra — not infra itself)

### Confirmed Context

| Decision | Choice | Notes |
|---|---|---|
| Layer classification | `terminus-platform` | Adds to existing Temporal worker service |
| Scope | Full migration + feature addition | Migrate existing TypeScript worker stubs to Go; implement ReleaseWorkflow in Go |
| Language | Go 1.25 | Aligns with `fourdogs-central` (Go 1.25) and `terminus-inference-gateway` (Go 1.24); TypeScript was an oversight on the original worker scaffold |
| Temporal SDK | `go.temporal.io/sdk` | First-class Go support; replaces `@temporalio/sdk` |
| Task queue | `terminus-platform` | Locked — unchanged |
| Test runner | `go test` + `testify` | Standard Go testing; co-located `_test.go` files |
| Logging | `log/slog` (stdlib) | Matches `fourdogs-central` and `emailfetcher` patterns |
| Secrets | Vault KV v2 → ESO → k8s Secret → env vars | Locked — no secrets in code |
| Deployment | ArgoCD + Helm, same `temporal-worker` chart | Image name unchanged: `ghcr.io/electricm0nk/terminus-platform-worker` |

### Migration Note (Existing TypeScript Worker)

The existing `terminus.platform` TypeScript service (`services/temporal/`) is **entirely stub code** — `DailyBriefingWorkflow` performs only payload validation, and `briefingActivities` returns a hardcoded placeholder. There is no production logic to preserve. This initiative replaces the TypeScript scaffold with a Go module as part of the same body of work.

The TypeScript-specific directories (`services/`, `shared/`, `node_modules/`, `coverage/`) and files (`package.json`, `package-lock.json`, `tsconfig.json`) are removed. The Helm chart, k8s manifests, and Dockerfile location are updated.

### Dependency Map

- **Prerequisite:** `terminus-platform-temporal` initiative complete — worker deployed and healthy
- **Prerequisite:** Temporal server running in k3s cluster
- **Prerequisite:** Semaphore has access to Temporal CLI (`temporal workflow start`)
- **In-scope:** Migrate TypeScript worker stubs → Go (clean rewrite; no logic to preserve)
- **In-scope:** `ReleaseWorkflow` Go implementation
- **In-scope:** `ReleaseInput` / `ReleaseResult` Go types (`internal/types/release.go`)
- **In-scope:** Go Dockerfile replacing Node.js Dockerfile
- **Out of scope:** Ansible playbook authoring (lives in `terminus.infra`)
- **Out of scope:** Semaphore task template setup (operator task)
- **Out of scope:** ArgoCD webhook infrastructure (polling chosen instead)

---

## Core Architectural Decisions (Step 4)

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Language migration: TypeScript → Go (already resolved above)
- Workflow trigger mechanism — Semaphore CLI dispatch
- DB provisioning activity — caller-controlled via `provisionDb` flag
- ArgoCD sync detection — polling with `activity.RecordHeartbeat`

**Important Decisions (Shape Architecture):**
- Workflow input schema — typed Go struct, serialized as JSON by Temporal
- Activity error handling — `temporal.NewNonRetryableApplicationError` for unrecoverable states
- Go module naming — `github.com/electricm0nk/terminus-platform`

**Deferred Decisions (Post-MVP):**

| Decision | Deferred Until |
|---|---|
| Multi-environment support (`env` beyond `dev`) | When second environment is introduced |
| Parallel service releases | After baseline single-service flow is stable |
| Rollback workflow (`RollbackWorkflow`) | After first production incident requiring it |
| Workflow history archival | After homelab storage review |

### Workflow Trigger

| Decision | Choice | Rationale |
|---|---|---|
| Trigger mechanism | `temporal workflow start` CLI in Semaphore task | Zero additional code; Temporal CLI is a Go binary — trivially available in Semaphore runner |
| Trigger point | End of Semaphore docker build+push task | Build must succeed before release workflow starts |
| CLI command pattern | `temporal workflow start --task-queue terminus-platform --type ReleaseWorkflow --input '{"service":"...","sha":"...","env":"dev","provisionDb":false}'` | Explicit, auditable, matches Temporal conventions |
| Workflow ID pattern | `release-{service}-{sha[:8]}` | Unique per release; prevents duplicate concurrent releases for same SHA |
| Workflow ID reuse policy | `ALLOW_DUPLICATE_FAILED_ONLY` | Re-triggers permitted only after a failed run; prevents double-release on success |

### Data Architecture

| Decision | Choice | Rationale |
|---|---|---|
| Workflow input type | `ReleaseInput` Go struct | Serialized as JSON by Temporal; typed; no untyped `interface{}` in workflow signatures |
| DB provisioning control | `ProvisionDB bool` in `ReleaseInput` | Caller (Semaphore task) knows whether this is a first-deploy; avoids fragile idempotency logic in the activity |
| Workflow result type | `ReleaseResult` struct with `Status`, `CompletedAt`, `SmokeTestPassed` | Structured output; visible in Temporal UI and queryable via CLI |
| Activity outputs | Each activity returns typed result; workflow assembles into `ReleaseResult` | Consistent with Go `go.temporal.io/sdk` activity patterns |
| `ModulePayload` Go equivalent | `types.ModulePayload[S any]` generic struct | Go 1.18+ generics; preserves the typed payload contract from the TypeScript original |

### Authentication & Security

| Decision | Choice | Rationale |
|---|---|---|
| Semaphore invocation | Semaphore REST API (`POST /api/v1alpha1/project/{id}/tasks`) | Programmatic trigger from activity; keeps Ansible on the Semaphore runner — no Ansible in worker container |
| Semaphore API token | Vault KV v2 → ESO → k8s Secret → env var | Consistent with all other credential delivery |
| ArgoCD API token | Vault KV v2 → ESO → k8s Secret → env var | Same pattern |
| Secrets in workflow input | Forbidden — Temporal serializes input to history | Service name, SHA, and flags only — never tokens or passwords |

### API & Communication

| Decision | Choice | Rationale |
|---|---|---|
| HTTP client | `net/http` stdlib | No external dependency needed for simple REST calls |
| Semaphore invocation | Fire-and-poll: `POST /tasks` → poll `GET /tasks/{id}` until terminal state | Consistent across Ansible and smoke test activities |
| ArgoCD sync detection | Poll `GET /api/v1/applications/{name}` heartbeat loop | Simple, no webhook infra; `activity.RecordHeartbeat` prevents timer timeout |
| ArgoCD poll interval | 15 seconds (`time.Sleep(15 * time.Second)`) | Balances responsiveness against API load |
| ArgoCD poll timeout | 10 minutes via `StartToCloseTimeout` on activity | Covers normal sync + image pull time; `NewNonRetryableApplicationError` on timeout |
| Smoke test invocation | Semaphore REST API — same fire-and-poll as Ansible activities | Consistent invocation model |

### Infrastructure & Deployment

| Decision | Choice | Rationale |
|---|---|---|
| Worker deployment | Existing `temporal-worker` Helm chart — no chart changes | Same image name: `ghcr.io/electricm0nk/terminus-platform-worker` |
| Dockerfile | New Go multi-stage Dockerfile at repo root | Replaces `services/temporal/Dockerfile`; Go binary in `FROM scratch` or `gcr.io/distroless/static` runtime |
| Worker image rebuild | Existing CI pipeline handles on merge to main | Dockerfile path update may be needed in CI config |
| New k8s resources | Two new `ExternalSecret` resources: `release-semaphore-token`, `release-argocd-token` | Added to `k8s/` directory per existing pattern |
| Vault paths | `secret/terminus/default/semaphore/api-token`, `secret/terminus/default/argocd/api-token` | Follows established `secret/terminus/default/{service}/...` hierarchy |

---

## Implementation Patterns & Consistency Rules (Step 5)

All patterns align with `fourdogs-central` (the reference Go service). Typescript-era conventions are superseded.

### Naming Patterns

| Area | Rule | Example |
|---|---|---|
| Workflow function name | `PascalCase` function | `ReleaseWorkflow` |
| Activity function names | `PascalCase` functions | `SeedSecrets`, `ProvisionDatabase`, `WaitForArgoCDSync`, `RunSmokeTest` |
| Go files | `snake_case.go` | `release.go`, `release_test.go`, `daily_briefing.go` |
| Go packages | lowercase, single word | `package workflows`, `package activities`, `package types` |
| Input/output types | `{WorkflowName}Input` / `{WorkflowName}Result` | `ReleaseInput`, `ReleaseResult` |
| Workflow ID | `release-{service}-{sha[:8]}` | `release-fourdogs-central-5aaca2d3` |
| k8s ExternalSecret names | `release-{purpose}` | `release-semaphore-token`, `release-argocd-token` |
| Vault paths | `secret/terminus/default/{service}/api-token` | `secret/terminus/default/semaphore/api-token` |
| Env var names | `SCREAMING_SNAKE_CASE` | `SEMAPHORE_API_TOKEN`, `ARGOCD_API_TOKEN`, `SEMAPHORE_PROJECT_ID` |

### Process Patterns

| Area | Rule |
|---|---|
| Activity retry policy | Default Temporal retry policy unless explicit override with documented justification |
| ArgoCD poll | Must call `activity.RecordHeartbeat(ctx, progress)` on every poll cycle |
| Unrecoverable errors | Return `temporal.NewNonRetryableApplicationError(msg, errType, cause)` — never return plain `errors.New` for terminal failures |
| Semaphore task trigger | Fire-and-poll: POST to trigger, then poll `GET /tasks/{id}` until `status == "finished"` or `"failed"` |
| `ProvisionDB: false` | `ProvisionDatabase` activity is skipped entirely — no no-op call; workflow branches on flag |
| Workflow determinism | All external calls in activities; workflow uses only `workflow.Sleep`, `workflow.ExecuteActivity`, `workflow.Now`; no `time.Now()` or `os.Getenv` in workflow code |
| Logging in activities | `slog.Info(...)` / `slog.Error(...)` with structured key/value pairs — no `fmt.Println` |
| Logging in workflow | None — workflows must be deterministic; use activity for any logging side-effects |
| Config from env | All URLs, tokens, queue names read via `os.Getenv` in `main.go` or activity constructors — never hardcoded |

### Anti-Patterns (Forbidden)

- Credentials in workflow input payload (stored in Temporal history — never safe)
- Hardcoded Vault paths, Semaphore project IDs, or ArgoCD URLs
- `time.Now()`, `os.Getenv`, `http.Get`, or any I/O inside a workflow function
- Polling without `activity.RecordHeartbeat(ctx, ...)` inside a long-running activity
- `MaximumAttempts: 1` on any activity without documented justification
- Ansible in the worker container (use Semaphore REST API to trigger it)
- `interface{}` in workflow/activity signatures (use typed structs)

### Mandatory Rules

- All existing and new tests pass 100% before story is ready
- TDD: `_test.go` scaffolded before implementation
- No untyped workflow inputs (`interface{}` or `map[string]any` in signatures forbidden)
- `slog.Info`/`slog.Error` with structured key/value pairs on all activity entry/exit and errors
- 80% line coverage minimum; 100% on `ReleaseInput` / `ReleaseResult` type validation

---

## Project Structure & Boundaries (Step 6)

### Repository Transformation

The existing npm workspace structure is replaced entirely by a Go module. The Helm chart and k8s manifests are retained with path updates.

**Removed (TypeScript scaffold):**
```
services/                    ← entire directory removed
shared/                      ← entire directory removed
node_modules/                ← removed
coverage/                    ← removed
package.json                 ← removed
package-lock.json            ← removed
tsconfig.json                ← removed
```

**New Go module structure:**
```
terminus.platform/
│
├── go.mod                              ← NEW: module github.com/electricm0nk/terminus-platform
├── go.sum                              ← NEW
│
├── cmd/
│   └── worker/
│       └── main.go                     ← NEW: replaces services/temporal/src/worker.ts
│
├── internal/
│   ├── types/
│   │   ├── module.go                   ← NEW: ModulePayload[S any], RunContext (translated from module.ts)
│   │   └── release.go                  ← NEW: ReleaseInput, ReleaseResult, ServiceTarget
│   ├── workflows/
│   │   ├── daily_briefing.go           ← NEW: replaces dailyBriefingWorkflow.ts (stub migration)
│   │   ├── daily_briefing_test.go      ← NEW
│   │   ├── release.go                  ← NEW: ReleaseWorkflow implementation
│   │   └── release_test.go             ← NEW
│   └── activities/
│       ├── briefing.go                 ← NEW: replaces briefingActivities.ts (stub migration)
│       ├── briefing_test.go            ← NEW
│       ├── release.go                  ← NEW: SeedSecrets, ProvisionDatabase, WaitForArgoCDSync, RunSmokeTest
│       └── release_test.go             ← NEW
│
├── Dockerfile                          ← NEW: Go multi-stage; replaces services/temporal/Dockerfile
│
├── helm/                               ← RETAINED (no changes)
│   ├── temporal-server/
│   └── temporal-worker/
│
└── k8s/                                ← UPDATED: two new ExternalSecret manifests added
    ├── external-secret-release-semaphore.yaml   ← NEW
    └── external-secret-release-argocd.yaml      ← NEW
```

### Architectural Boundaries

**`ReleaseWorkflow` boundary** (`internal/workflows/release.go`):
- Accepts `ReleaseInput` from Temporal task queue (JSON-deserialized by SDK)
- Sequences activities — owns execution policy, retry config, timeout decisions
- Branches on `input.ProvisionDB` to skip `ProvisionDatabase`
- Returns `ReleaseResult` on completion; fails with `temporal.NewNonRetryableApplicationError` on unrecoverable errors
- Zero I/O — pure workflow logic; never calls `os.Getenv`, `time.Now()`, or `http`

**Activity boundary** (`internal/activities/release.go`):
- `SeedSecrets`: POST Semaphore task → poll to completion (triggers `seed-{service}-secrets.yml`)
- `ProvisionDatabase`: POST Semaphore task → poll to completion (triggers `db-provision.yml`); skipped entirely if `ProvisionDB: false`
- `WaitForArgoCDSync`: Poll ArgoCD API with `activity.RecordHeartbeat`; returns when `health.status == Healthy` and `sync.status == Synced`
- `RunSmokeTest`: POST Semaphore task → poll to completion (triggers `verify-{service}-deploy.yml`)

**Semaphore boundary** (not owned by this repo):
- Ansible playbooks live in `terminus.infra`
- Semaphore task templates are operator-configured — not in code

**Out of bounds for this initiative:**
- ArgoCD Application definitions (live in `terminus.infra`)
- Ansible playbook authoring (live in `terminus.infra`)
- Vault secret seeding of `semaphore/api-token` and `argocd/api-token` (operator prerequisite)

### Integration Points

**Data flow — release execution:**
```
Semaphore CI (build + push Go binary image to GHCR)
  → temporal workflow start (CLI) with ReleaseInput JSON
  → Temporal server enqueues on terminus-platform task queue
  → Worker (Go binary) executes ReleaseWorkflow
  → SeedSecrets → Semaphore REST API → ansible: seed-{service}-secrets.yml
  → [if ProvisionDB] ProvisionDatabase → Semaphore REST API → ansible: db-provision.yml
  → ArgoCD detects image tag change in Helm values → begins sync
  → WaitForArgoCDSync → poll ArgoCD REST API until Healthy + Synced (heartbeat every 15s)
  → RunSmokeTest → Semaphore REST API → ansible: verify-{service}-deploy.yml
  → ReleaseWorkflow complete → ReleaseResult in Temporal visibility (PostgreSQL)
```

**Prerequisite credential flow:**
```
Operator seeds Vault:
  secret/terminus/default/semaphore/api-token
  secret/terminus/default/argocd/api-token
  → ESO ExternalSecrets sync to k8s Secrets in temporal namespace
  → Worker Deployment mounts as env vars (SEMAPHORE_API_TOKEN, ARGOCD_API_TOKEN)
  → Activities read from os.Getenv (never hardcoded; never in workflow function)
```

---

## Architecture Validation Results (Step 7)

### Coherence Validation ✅

**Decision Compatibility:**
All technology choices are internally consistent and match the established stack. Go 1.25 + `go.temporal.io/sdk` is a first-class, actively maintained combination. `net/http` stdlib for Semaphore and ArgoCD REST calls needs no external HTTP library. `log/slog` is stdlib — no new dependency. `activity.RecordHeartbeat` is the idiomatic Go SDK pattern for exactly this polling scenario. ESO → k8s Secret → env var is unchanged from every other service credential pattern in this stack.

**Pattern Consistency:**
Go naming conventions are internally consistent (`PascalCase` workflow/activity functions, `snake_case` files, lowercase packages). The `ModulePayload[S any]` generic struct preserves the typed payload contract from the TypeScript original while being idiomatic Go 1.18+. `temporal.NewNonRetryableApplicationError` mirrors the TypeScript `ApplicationFailure.nonRetryable` semantics exactly. The fire-and-poll Semaphore API pattern is consistent across all three activities that use it.

**Structure Alignment:**
The Go module layout (`cmd/worker/`, `internal/workflows/`, `internal/activities/`, `internal/types/`) is the standard idiomatic Go project layout. Boundaries are clean — workflow code in `internal/workflows/`, external I/O in `internal/activities/`, types in `internal/types/`. The `cmd/worker/main.go` entrypoint is a clean seam that owns all config loading via `os.Getenv`, keeping activities testable without environment setup.

**Language Migration Coherence:**
The TypeScript-to-Go migration is safe because the existing TypeScript code is entirely stubs with no production logic. `DailyBriefingWorkflow` only validates payload shape. `briefingActivities` returns a hardcoded placeholder. The migration is a clean rewrite, not a port — no business logic is at risk.

---

### Requirements Coverage ✅

**Functional Requirements:**
| Requirement | Architectural Support |
|---|---|
| Vault seeding | `SeedSecrets` → Semaphore REST API → Ansible playbook |
| DB provisioning (conditional) | `ProvisionDatabase`, skipped entirely when `ProvisionDB: false` |
| ArgoCD sync wait | `WaitForArgoCDSync` with 15s poll, 10min `StartToCloseTimeout`, mandatory `RecordHeartbeat` |
| Smoke test | `RunSmokeTest` → Semaphore REST API, consistent with other activities |
| Semaphore trigger | `temporal workflow start` CLI at end of Semaphore build task |
| Idempotent release | `ALLOW_DUPLICATE_FAILED_ONLY` workflow ID reuse policy |
| Migrate existing TypeScript worker | Full rewrite to Go; TypeScript scaffold removed |

**Non-Functional Requirements:**
- **Security:** Credentials never in workflow input; Vault KV → ESO → env var chain; enforced by anti-pattern rule; no Ansible in worker container
- **Reliability:** Default Temporal retry on activities; `NewNonRetryableApplicationError` on genuine terminal failures; 10-minute ArgoCD timeout prevents infinite wait
- **Observability:** Temporal UI provides full workflow history; `slog.Info`/`slog.Error` on all activity entry/exit and errors with structured key/value pairs
- **Determinism:** Enforced — workflow function is pure; all I/O in activities; `time.Now()`, `os.Getenv`, `http` forbidden in workflow code (Go SDK panics on non-deterministic calls in workflow context anyway — a compile-time safety net)
- **Language consistency:** Go aligns `terminus.platform` with `fourdogs-central` and `terminus-inference-gateway` — no more polyglot penalty for new contributors

---

### Implementation Readiness ✅

**Decision Completeness:** All critical decisions are documented with rationale and concrete examples. Go module name, version, SDK, logging, error types, HTTP client, and credential delivery path are all specified. The Dockerfile strategy (multi-stage Go build to distroless) is specified. No ambiguous decisions remain open.

**Structure Completeness:** Every new file has an exact path. The two existing files this work affects (`helm/temporal-worker/values.yaml` — unchanged, Dockerfile — replaced) are identified. Both k8s ESO manifests are named. The TypeScript directories to be removed are listed explicitly.

**Pattern Completeness:** Naming conventions cover functions, files, packages, types, env vars, k8s resources, and Vault paths. Error handling specifies the exact Go SDK error constructor. Polling specifies mandatory `activity.RecordHeartbeat`. Logging specifies structured `slog` key/value pairs. The determinism rules include the note that the Go SDK itself enforces them at runtime, providing an additional safety net.

---

### Gap Analysis

**Critical Gaps:** None. All implementation-blocking questions are answered.

**Important Gaps (non-blocking — suitable for story-level AC):**
- Semaphore task template naming convention for Ansible jobs (e.g., `seed-fourdogs-central-secrets`, `db-provision-fourdogs-central`) — activities will construct these names; finalized in story AC
- `extraVars` payload passed to Ansible via Semaphore task API — implementation detail, not architectural
- Exact Semaphore project ID env var sourcing — operator-configured; `SEMAPHORE_PROJECT_ID` specified as env var name but value is runtime config

**Nice-to-Have:**
- Explicit retry override for `WaitForArgoCDSync` — heartbeat changes effective timeout model vs. default retry; likely `MaximumAttempts: 1` with long `StartToCloseTimeout` is correct
- Definition of `SmokeTestPassed: true` — what does the Ansible `verify-{service}-deploy.yml` report back?

---

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed (homelab, single-node Temporal, Go-first stack)
- [x] Technical constraints identified (Go locked, task queue locked, image name unchanged)
- [x] Cross-cutting concerns mapped (security, observability, determinism, language consistency)
- [x] Migration scope assessed (TypeScript stubs only — safe clean rewrite)

**✅ Architectural Decisions**
- [x] Critical decisions documented with rationale
- [x] Technology stack fully specified (Go 1.25, `go.temporal.io/sdk`, `log/slog`, `net/http`)
- [x] Integration patterns defined (Semaphore REST API, ArgoCD REST API, fire-and-poll)
- [x] Performance considerations addressed (poll interval, timeouts, heartbeat)
- [x] Migration decision documented and justified

**✅ Implementation Patterns**
- [x] Naming conventions established with examples (functions, files, packages, types, env vars)
- [x] Structure patterns defined (standard Go project layout)
- [x] Communication patterns specified (`net/http`, fire-and-poll)
- [x] Process patterns documented (heartbeat, nonRetryable, TDD, slog)

**✅ Project Structure**
- [x] Complete file delta defined (new files, updated files, removed files)
- [x] Component boundaries established
- [x] Integration points mapped with data flow
- [x] No new deployment infrastructure required (same Helm chart, same image name)

---

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High — Go is the established stack language; the TypeScript worker was a scaffold with no business logic; migration is low-risk; all patterns have Go analogues with clear precedent in `fourdogs-central`.

**Key Strengths:**
- Language consistency: `terminus.platform` now matches the rest of the stack
- Go binary in distroless container: smaller image, faster startup, no runtime dependency on Node.js or npm
- `activity.RecordHeartbeat` requirement is explicit — avoids the most common Temporal long-activity pitfall
- Fire-and-poll via Semaphore API keeps Ansible execution on the Semaphore runner — no credential leakage into the worker container
- `ProvisionDB` flag design avoids fragile idempotency logic in the activity
- TypeScript migration is zero-risk: all stubs, no real logic to lose

**Areas for Future Enhancement:**
- Rollback workflow (`RollbackWorkflow`) — no automated rollback if ArgoCD sync fails post-secrets-seeding
- Multi-environment support (`Env: staging|prod`) — currently only `dev` in scope
- Parallel service releases — currently single-service per workflow instance
- `WaitForArgoCDSync` retry override — likely needs `MaximumAttempts: 1` clarified in story AC

### Implementation Handoff

**Agent Guidelines:**
- Follow all architectural decisions exactly as documented
- All Go patterns from `fourdogs-central` apply — it is the reference implementation
- Workflow function is pure — Go SDK will panic if non-deterministic calls are made inside it; trust the SDK
- All `os.Getenv` calls belong in `cmd/worker/main.go` or activity constructors — not deep in activity logic
- Start with types (`internal/types/`), then activities (with tests), then workflow (with tests), then wire up `cmd/worker/main.go`

**First Implementation Priority:**
1. `go.mod` + project scaffold
2. `internal/types/release.go` — `ReleaseInput`, `ReleaseResult`
3. `internal/activities/release.go` + `release_test.go`
4. `internal/workflows/release.go` + `release_test.go`
5. `cmd/worker/main.go` — register and start worker
6. New Go `Dockerfile`
7. `internal/types/module.go` + `internal/workflows/daily_briefing.go` — migrate stubs
8. k8s ESO manifests
9. Remove TypeScript scaffold
