---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-04-11'
inputDocuments:
  - docs/cicd.md
  - docs/terminus/architecture.md
  - docs/terminus/platform/project-context.md
  - docs/terminus/platform/releaseorchestrator/architecture.md
  - docs/terminus/platform/temporal/architecture.md
workflowType: architecture
initiative: terminus-platform-deliverycontrol
track: tech-change
date: '2026-04-11'
---

# Architecture — Terminus Delivery Control

_Collaborative architecture document for `terminus-platform-deliverycontrol` (tech-change track)._
_Sections are appended through working sessions._

---

## Problem Statement

### Context

The Terminus platform has a documented, well-defined release contract:

```
Operator → Semaphore (release template)
         → Ansible (start-release-workflow.yml)
         → Temporal ReleaseWorkflow
              ├── SeedSecrets      → Ansible/Vault
              ├── ProvisionDatabase → Ansible (if needed)
              ├── WaitForArgoCDSync → ArgoCD API poll
              └── RunSmokeTest     → Ansible verify
```

This contract is sound. Temporal provides durable orchestration, Semaphore provides a human entry-point, and ArgoCD provides GitOps delivery. The Temporal `ReleaseWorkflow` exists and is tested. Semaphore templates for seeding secrets and running smoke tests are defined for the portal service.

The problem is not the contract. The problem is the segment *before* it.

---

### The Gap

**Between "an image is built and pushed to GHCR" and "the release contract begins" — there is nothing.**

This gap has three distinct failure modes:

#### 1. Change Detection — Undefined

When a service's code changes, a new container image is built and pushed to GHCR. No mechanism currently exists to:
- Detect that a new image has been published
- Assess whether that image is a candidate for release
- Determine if a release should be triggered and for which environment

Each service currently bridges this gap differently and informally:
- **terminus-portal**: Static site with no image; direct `index.html` changes; operator must notice the PR merge and manually trigger a release.
- **inference-gateway**: `latest` tag pinned in ArgoCD; ArgoCD auto-syncs when the tag resolves to a new digest (implicit, non-auditable, no Temporal involvement).
- **fourdogs-central**: GitHub Actions mutates Helm values in-repo after every build push (bypasses release contract entirely).

None of these are the documented model.

#### 2. Release Initiation — Ungoverned

Even when an operator decides a release is warranted, there is no defined framework for:
- What services should require an explicit human approval to release vs what can be automatic
- Where this decision is expressed (environment config? service config? policy?)
- How a pending release communicates its readiness to the operator

The result is that every release is invisible until the operator decides to act — and there is no record of what triggered the decision.

#### 3. Manifest Promotion — Absent

ArgoCD applies what is in git. For ArgoCD to deploy a specific image version, that version must be written into the Helm values or application spec committed to the infra repo. Currently:
- Services using `:latest` never require a manifest change (ArgoCD will pick up the new digest on sync), but this bypasses version pinning and makes rollback difficult.
- Services with explicit version pins in Helm values have no automated mechanism to update those pins after a new image is pushed.

Without a manifest promotion step, the release contract's `WaitForArgoCDSync` activity is waiting for a change that may never be written to git — or may have been written directly to git by a GitHub Actions workflow, bypassing Temporal entirely.

---

### What Is Not the Problem

This initiative is **not** about replacing the Temporal `ReleaseWorkflow`. That contract is correct and must remain in place. Any solution must feed *into* it, not around it.

This initiative is **not** a CI tooling question. Whether GitHub Actions uses cloud runners or local runners is a deployment choice for CI tasks (build, test, push). The delivery control problem begins *after* the push.

---

### What Needs to Be Decided

This initiative must produce a coherent design for:

1. **Change detection mechanism** — How does the system learn that a new release candidate exists? Options include: polling GHCR/git, webhook from CI, events from a CI run, durable watch activity in Temporal, or a combination.

2. **Release intent expression** — How does an operator or an automated policy express "this artifact is ready to release to environment X"? What is the data model? Where is it stored?

3. **Approval model** — Which services auto-release on detection? Which require operator approval? How is this policy declared and enforced?

4. **Manifest promotion** — How are Helm values or ArgoCD app specs updated with a specific image tag before `WaitForArgoCDSync` has anything to wait for? Who writes the commit? When? Under what conditions?

5. **Integration boundary** — At what exact point does the new mechanism hand off to the existing Temporal `ReleaseWorkflow`? What is the contract at that boundary?

---

### Scope

| In Scope | Out of Scope |
|---|---|
| Change detection design for all Terminus-domain services | CI pipeline internals (build steps, test gates) |
| Release intent and approval model | Temporal `ReleaseWorkflow` internals (already defined) |
| GitOps manifest promotion mechanism | New service onboarding workflow |
| Integration boundary with Temporal `ReleaseWorkflow` | Multi-environment promotion topology (future) |
| Policy/config model for per-service approval requirements | External environments or cloud targets |

---

### Success Criteria

The initiative is complete when:
- Every Terminus-domain service has exactly one, clearly defined path from "image pushed" to "ArgoCD sync confirmed"
- That path passes through Temporal `ReleaseWorkflow` for every service
- The approval model is declared in service config, not improvised per-deployment
- Manifest promotion is automated and auditable — no manual git commits to promote an image version
- Rollback is possible by reverting a single, identifiable commit

---

## Project Context Analysis

### Revised Release Contract

This initiative revises the release contract. The previous contract required a human to open Semaphore and manually trigger the release template. The new contract is:

> **A PR merge IS the release approval. No further human action is required or permitted after merge.**

| Branch merge | Target environment | Environment namespace |
|---|---|---|
| `feature/*` → `develop` | Development | `{service}-dev` |
| `develop` → `main` | Production | `{service}` |

Semaphore retains its role as the **break-glass emergency path** only. It is not the normal release entry point under this architecture.

### Existing Platform Inventory

| Component | State | Notes |
|---|---|---|
| `ReleaseWorkflow` (Go) | ✅ Complete | Takes `ReleaseInput{ServiceName, ProvisionDB}` — extended in this initiative |
| `SeedSecrets` activity | ✅ Complete | Calls `seed-{ServiceName}-secrets` Semaphore template |
| `WaitForArgoCDSync` activity | ✅ Complete | Polls ArgoCD API with heartbeat |
| `RunSmokeTest` activity | ✅ Complete | Calls `verify-{ServiceName}-deploy` Semaphore template |
| `terminus.platform` worker | ✅ Deployed | ArgoCD + Helm, `ghcr.io/electricm0nk/terminus-platform-worker` |
| GitHub Actions runners | ⚠️ Cloud only | Cannot reach `*.trantor.internal` — gap; resolved by this initiative |
| Manifest promotion | ❌ Absent | Image tags not automatically committed to `terminus.infra` |
| Environment split | ❌ Absent | No `dev` vs `prod` ArgoCD apps or values files |
| Self-hosted k3s runner | ❌ Absent | Required for network-adjacent CI steps |

### Integration Complexity

High integration surface: GitHub Actions, GHCR, `terminus.infra` git, Temporal, ArgoCD, k3s, Vault/ESO. Primary risk is state consistency between the manifest promotion commit and the Temporal workflow start — both must succeed or neither should proceed.

---

## Core Architectural Decisions

### Decision 1 — Approval Model

| Decision | Choice | Rationale |
|---|---|---|
| Human approval mechanism | **Git PR merge** | A merged PR is the explicit, auditable approval signal. No separate approval UI. |
| Auto-release trigger | **GitHub Actions** `on: pull_request` closed+merged | Native to every service repo; no new infrastructure for event detection |
| Semaphore role | **Break-glass emergency only** | Normal release path bypasses Semaphore; operator can still use it to force-release if automation fails |
| Per-service approval override | **Out of scope** for this initiative | All services auto-release on merge; per-service exemptions are a future policy feature |

### Decision 2 — CI Runner Strategy

| Decision | Choice | Rationale |
|---|---|---|
| Runner type | **Self-hosted k3s runner** | Cloud runners cannot reach `*.trantor.internal`; k3s runner has in-cluster network access |
| Runner placement | `terminus-infra` namespace in k3s | Infra concern; co-located with ArgoCD, network-adjacent to Temporal frontend |
| Runner label | `self-hosted`, `trantor-internal` | Jobs targeting internal cluster steps use `runs-on: [self-hosted, trantor-internal]` |
| Build/test steps | **Cloud runners** (no label) | Build, test, push to GHCR do not need cluster access; use GitHub-hosted runners for speed |
| Runner registration | GitHub Actions runner token via Vault ESO | Secret management follows existing pattern; no plaintext tokens in git |
| Runner deployment | Single `Deployment` in `terminus.infra` | Managed like any other k8s workload; ArgoCD-deployed |

### Decision 3 — Manifest Promotion

| Decision | Choice | Rationale |
|---|---|---|
| Promotion mechanism | **GitHub Actions step (local runner)** | Fastest to implement; three lines of shell; auditable commit in `terminus.infra` |
| Image tag strategy | **Git SHA** (`ghcr.io/{org}/{service}:{sha}`) | Immutable, traceable to exact commit; `latest` is forbidden after this initiative |
| Values file target | `terminus.infra` `values-dev.yaml` or `values.yaml` per environment | ArgoCD apps will reference environment-specific values files |
| Commit message convention | `chore(release): promote {service}:{sha[:8]} to {env}` | Readable audit trail in `terminus.infra` git log |
| Promotion atomicity | Single `values.yaml` file per service per environment | Multi-file promotion deferred; single-file is atomic per git commit |
| Deferred upgrade | Migrate to `ManifestPromotionActivity` in Temporal worker when durability matters | When first production incident demonstrates the gap; recorded as deferred decision |

### Decision 4 — Environment Model

| Decision | Choice | Rationale |
|---|---|---|
| Environment count | **Two**: `dev`, `prod` | Single k3s cluster; namespaces provide isolation |
| Namespace convention | `{service}-dev` (dev) / `{service}` (prod) | Consistent with existing prod namespaces; dev suffix is unambiguous |
| ArgoCD app convention | `{service}-dev.yaml` / `{service}.yaml` in `platforms/k3s/argocd/apps/` | Mirrors existing pattern; adds `-dev` variant |
| ArgoCD `targetRevision` | `develop` branch for dev apps; `HEAD` (`main`) for prod apps | Branch-tracking in `terminus.infra`; ArgoCD polls for new commits |
| Values files | `values-dev.yaml` / `values.yaml` per service | Dev values override image tag only; prod values are the canonical values |
| DNS convention | `{service}-dev.trantor.internal` (dev) / `{service}.trantor.internal` (prod) | Requires one additional DNS A-record per service on Synology NAS |

### Decision 5 — ReleaseInput Extension

The existing `ReleaseInput` type is extended. **Backwards compatible** — existing Semaphore-triggered runs continue to work.

```go
// internal/types/release.go (extended)
type ReleaseInput struct {
    ServiceName string `json:"ServiceName"`             // existing — unchanged
    ProvisionDB bool   `json:"ProvisionDB"`             // existing — unchanged
    Environment string `json:"Environment"`             // NEW: "dev" | "prod"; default "dev"
    ImageTag    string `json:"ImageTag"`                // NEW: full tag e.g. "sha-5aaca2d3"; for audit
}
```

Activity Semaphore template routing adapts to environment:
- `SeedSecrets` → `seed-{ServiceName}-{env}-secrets` (or `seed-{ServiceName}-secrets` for shared secrets)
- `RunSmokeTest` → `verify-{ServiceName}-{env}-deploy`

**Note:** Secret seeding is environment-agnostic in this homelab (single Vault instance, single secret path). Semaphore template names for seed operations remain unchanged. Smoke test templates are environment-aware.

### Decision 6 — Workflow ID Scheme (Revised)

Revised from `release-{service}-{sha[:8]}` to include environment:

```
release-{service}-{env}-{sha[:8]}
```

Example: `release-fourdogs-central-dev-5aaca2d3`

Prevents collision when the same SHA is promoted to both environments (dev first, then prod).

### Deferred Decisions

| Decision | Deferred Until |
|---|---|
| `ManifestPromotionActivity` in Temporal (replace GH Actions git commit) | After first production incident exposing partial-promotion failures |
| `RollbackWorkflow` | After first production incident requiring it |
| Per-service approval policy (opt-out of auto-release) | When a service owner requests it |
| Parallel service releases | After baseline single-service flow is stable |
| Multi-cluster environment topology | If/when a second k3s cluster is introduced |

---

## Component Structure

### New Components (this initiative)

```
terminus.infra/
  platforms/k3s/
    k8s/
      actions-runner/
        runner-deployment.yaml            ← NEW: self-hosted Actions runner Deployment
        runner-external-secret.yaml       ← NEW: runner registration token from Vault
        runner-rbac.yaml                  ← NEW: ServiceAccount + ClusterRoleBinding
    argocd/apps/
      actions-runner.yaml                 ← NEW: ArgoCD Application for runner Deployment
      {service}-dev.yaml                  ← NEW: per-service dev ArgoCD Application (×N services)
    helm/{service}/
      values-dev.yaml                     ← NEW: per-service dev values (image.tag only)

terminus.platform/ (internal/types/release.go, internal/activities/release.go)
  internal/types/release.go               ← MODIFIED: add Environment, ImageTag fields
  internal/activities/release.go          ← MODIFIED: env-aware Semaphore template routing
  internal/activities/release_test.go     ← MODIFIED: tests for new routing

{service-repo}/.github/workflows/
  release.yml                             ← NEW (per service): build → push → promote → trigger
```

### Self-Hosted Runner Architecture

```
GitHub Actions (cloud)
  job: build-and-push
    runs-on: ubuntu-latest              ← cloud runner
    - checkout
    - docker build → ghcr.io/{org}/{service}:{sha}
    - docker push

  job: promote-and-release
    runs-on: [self-hosted, trantor-internal]   ← k3s runner
    needs: build-and-push
    - git clone terminus.infra
    - update values-{env}.yaml: image.tag = {sha}
    - git commit + push → terminus.infra develop|main
    - temporal workflow start (via kubectl exec into temporal-admintools)
          --type ReleaseWorkflow
          --input '{"ServiceName":"...","ProvisionDB":false,"Environment":"dev|prod","ImageTag":"sha-..."}'
          --workflow-id "release-{service}-{env}-{sha[:8]}"
          --id-reuse-policy ALLOW_DUPLICATE_FAILED_ONLY
```

### Release Flow — End to End (New Contract)

```
Developer opens PR → feature/* → develop
  ↓ PR reviewed + merged
GitHub Actions: on[pull_request: closed, branches: [develop]]
  if: github.event.pull_request.merged == true
  ├── [cloud runner] build image → push ghcr.io/{org}/{service}:{sha}
  └── [k3s runner]   commit image.tag={sha} → terminus.infra develop
                     temporal workflow start → ReleaseWorkflow{env: dev}
                       ├── SeedSecrets → seed-{service}-secrets Semaphore template
                       ├── WaitForArgoCDSync → ArgoCD polls terminus.infra develop
                       └── RunSmokeTest → verify-{service}-dev-deploy Semaphore template

Developer opens PR → develop → main
  ↓ PR reviewed + merged
GitHub Actions: on[pull_request: closed, branches: [main]]
  if: github.event.pull_request.merged == true
  ├── [cloud runner] (image already exists at {sha}) — no rebuild needed
  └── [k3s runner]   commit image.tag={sha} → terminus.infra main
                     temporal workflow start → ReleaseWorkflow{env: prod}
                       ├── SeedSecrets → seed-{service}-secrets Semaphore template
                       ├── WaitForArgoCDSync → ArgoCD polls terminus.infra main
                       └── RunSmokeTest → verify-{service}-deploy Semaphore template
```

**Note on prod image:** The image pushed on the `develop` merge is reused for production. No rebuild. The `main` merge workflow uses the SHA from the tip of `develop` — the same artifact that passed dev smoke tests. This requires the workflow to resolve the SHA rather than rebuild.

### ArgoCD Application Split

Each service goes from one ArgoCD application to three (infra + app + ingress becomes six with dev variants). The pattern is additive — existing prod apps are unchanged.

```yaml
# Example: fourdogs-central-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: fourdogs-central-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/electricm0nk/terminus.infra.git
    targetRevision: develop                    # ← tracks develop branch
    path: platforms/k3s/helm/fourdogs-central
    helm:
      valueFiles:
        - values-dev.yaml                      # ← dev-specific values
  destination:
    namespace: fourdogs-central-dev            # ← dev namespace
```

---

## Coding Standards & Patterns

### Naming Conventions (extending releaseorchestrator conventions)

| Entity | Convention | Example |
|---|---|---|
| GitHub Actions workflow file | `release.yml` | `.github/workflows/release.yml` |
| GitHub Actions job (cloud) | `build-and-push` | — |
| GitHub Actions job (local) | `promote-and-release` | — |
| ArgoCD dev app file | `{service}-dev.yaml` | `fourdogs-central-dev.yaml` |
| Dev namespace | `{service}-dev` | `fourdogs-central-dev` |
| Dev values file | `values-dev.yaml` | `helm/fourdogs-central/values-dev.yaml` |
| Smoke test template (dev) | `verify-{service}-dev-deploy` | `verify-fourdogs-central-dev-deploy` |
| Smoke test template (prod) | `verify-{service}-deploy` | `verify-fourdogs-central-deploy` |
| Temporal workflow ID | `release-{service}-{env}-{sha[:8]}` | `release-fourdogs-central-dev-5aaca2d3` |
| Runner k8s Deployment | `actions-runner` | namespace: `actions-runner` |
| Vault path (runner token) | `secret/terminus/default/github/runner-token` | — |

### Mandatory Rules (inherited + extended)

- All existing Go tests pass 100% before any story is ready
- `ReleaseInput` extension is backwards-compatible — `Environment` defaults to `"dev"` if absent
- `ImageTag` is advisory (audit/logging only) — workflow never uses it to pull images
- Self-hosted runner secret rotation follows the Vault ESO pattern — no static tokens in k8s Secrets
- GitHub Actions workflows must fail fast: if manifest promotion commit fails, do NOT call `temporal workflow start`
- No image rebuilds in the `develop → main` promotion path — reuse the SHA artifact from the dev release

### Anti-Patterns (Forbidden)

- `:latest` image tag in any ArgoCD Application or values file after this initiative ships
- Cloud GitHub Actions runner for any step that reaches `*.trantor.internal`
- Direct `kubectl apply` or `helm install` in GitHub Actions — ArgoCD is the only delivery mechanism
- `temporal workflow start` before manifest promotion commit is confirmed pushed
- Separate `ReleaseWorkflow` invocations for the same SHA in dev and prod that use different images

---

## Validation

### Implementation Readiness

| Prerequisite | Status |
|---|---|
| `terminus-platform-temporal` initiative complete | ✅ |
| `ReleaseWorkflow` Go implementation deployed | ✅ |
| `temporal-admintools` pod accessible in k3s | ✅ |
| Vault writable for new runner-token secret path | ✅ (existing Vault instance) |
| `terminus.infra` has SOPS/age key for infra runner commits | ⚠️ Runner needs git identity + push rights — GitHub deploy key required |
| Per-service `values-dev.yaml` files | ❌ New — part of this initiative |
| Per-service `-dev` ArgoCD Applications | ❌ New — part of this initiative |
| Per-service dev Semaphore smoke-test template | ❌ New — part of this initiative |
| Per-service dev DNS record on Synology NAS | ❌ Operator task per service |

### Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Manifest promotion commit succeeds, `temporal workflow start` fails | Medium | Medium | CI workflow explicitly checks exit code; failed Temporal start is a CI failure; no zombie promotion commits |
| ArgoCD syncs wrong commit (race between dev and prod promote) | Low | High | Separate ArgoCD apps track separate branches; no cross-contamination |
| Self-hosted runner becomes bottleneck (single pod, one job at a time) | Low | Low | Scale runner replica count if needed; initial homelab workload is minimal |
| Image built from `feature/*` is promoted to `develop` but has a bad SHA | Low | Low | SmokeTest catches it in dev before prod promotion is possible |
| Runner pod secret rotation causes release failure mid-run | Very Low | Low | Vault ESO renews automatically; runner restart resolves token refresh |
