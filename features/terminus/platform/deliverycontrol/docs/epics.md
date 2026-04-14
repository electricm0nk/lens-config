---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/platform/deliverycontrol/architecture.md
  - docs/cicd.md
  - docs/terminus/architecture.md
  - docs/terminus/platform/project-context.md
  - docs/terminus/platform/releaseorchestrator/architecture.md
  - docs/terminus/platform/temporal/architecture.md
workflowType: epics-and-stories
project_name: terminus-platform-deliverycontrol
user_name: electricm0nk
date: '2026-04-11'
status: complete
lastStep: 4
---

# terminus-platform-deliverycontrol - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for terminus-platform-deliverycontrol, decomposing the release-control architecture, CI/CD contract, and platform constraints into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: The system must treat a merged PR as the only normal release approval signal, with `develop` merges triggering dev release and `main` merges triggering production release.
FR2: The system must provide a per-service GitHub Actions `release.yml` workflow that triggers on merged PRs to `develop` and `main`.
FR3: The system must build and push an immutable container image tagged with the commit SHA to GHCR on a cloud GitHub Actions runner.
FR4: The system must run manifest promotion and Temporal release trigger steps on a self-hosted k3s GitHub Actions runner labeled for internal network access.
FR5: The system must promote releases by committing the resolved image tag to `terminus.infra` in `values-dev.yaml` for dev or `values.yaml` for prod.
FR6: The system must start the existing Temporal `ReleaseWorkflow` after manifest promotion using input that includes `ServiceName`, `ProvisionDB`, `Environment`, and `ImageTag`.
FR7: The system must use workflow IDs in the format `release-{service}-{env}-{sha[:8]}` with duplicate-failed-only reuse semantics.
FR8: The system must support two deployment environments in the same k3s cluster: `dev` in namespace `{service}-dev` and `prod` in namespace `{service}`.
FR9: The system must add dev ArgoCD applications, dev values files, and dev ingress/DNS patterns alongside the existing prod deployment pattern.
FR10: The system must route release activities through the existing release orchestration contract: `SeedSecrets`, optional `ProvisionDatabase`, `WaitForArgoCDSync`, and `RunSmokeTest`.
FR11: The system must make release activity routing environment-aware, including smoke test template naming and environment-specific release execution.
FR12: The system must keep Semaphore as a break-glass emergency path that can start the Temporal release manually when GitHub Actions is unavailable.
FR13: The system must deploy and configure a shared self-hosted GitHub Actions runner in k3s, with registration credentials delivered through Vault and ESO.
FR14: The system must preserve reuse of the same image artifact from dev to prod promotion rather than rebuilding on the `develop` to `main` merge path.

### NonFunctional Requirements

NFR1: The normal release path must require no human action after PR merge.
NFR2: The release approval and promotion path must be fully auditable through git history, workflow runs, and Temporal workflow IDs.
NFR3: Immutable SHA-based image tags are required; `latest` must not be used after this initiative ships.
NFR4: Manifest promotion must be revertible through a single identifiable git commit in `terminus.infra`.
NFR5: The release design must remain backward compatible with the existing Temporal `ReleaseWorkflow` and existing Semaphore-triggered runs.
NFR6: Any step that needs access to `*.trantor.internal` must execute on the self-hosted k3s runner, not a cloud runner.
NFR7: Secrets and runner credentials must follow the Vault KV to ESO to k8s Secret pattern; no static credentials in git or workflow inputs.
NFR8: GitHub Actions must fail fast if manifest promotion fails; it must not start Temporal after a failed promotion commit.
NFR9: The same artifact promoted to prod must be the one that passed dev verification; the prod path must not rebuild a different image.
NFR10: Break-glass Semaphore usage must remain explicitly secondary and exceptional.
NFR11: Implementation must address concurrent manifest promotions safely to avoid push conflicts or out-of-order releases.
NFR12: Environment naming and routing must come from one canonical mapping to prevent branch, namespace, ArgoCD app, values file, and smoke-test drift.

### Additional Requirements

- Extend `ReleaseInput` with `Environment` and `ImageTag` while keeping backwards compatibility for older manual starts.
- Update release activity routing so smoke tests are environment-aware and secret seeding remains compatible with shared secret patterns.
- Add `values-dev.yaml` per service and `-dev` ArgoCD app definitions that track the `develop` branch.
- Add dev DNS records using `{service}-dev.trantor.internal` and keep prod on `{service}.trantor.internal`.
- Create or update Semaphore templates for `seed-{service}-secrets`, `db-provision-{service}`, `verify-{service}-dev-deploy`, and `verify-{service}-deploy`.
- The self-hosted runner deployment lives in k3s and must use Vault-managed registration credentials.
- Manifest promotion and Temporal start form a coupled boundary; implementation must include retry or serialization behavior around `terminus.infra` writes.
- GitHub Actions is the primary CI/CD entry point; direct `kubectl apply` or `helm install` from workflows is forbidden because ArgoCD remains the only delivery mechanism.
- The develop-to-main promotion path must resolve and reuse the previously built SHA artifact instead of rebuilding.
- Release observability must make it obvious that GitHub Actions is the primary path and Semaphore is fallback only.
- No starter template is involved; this is an integration change across existing platform and infra assets.

### FR Coverage Map

FR1: Epic 1 - PR merge is the release approval
FR2: Epic 1 - per-service `release.yml` entry workflow
FR3: Epic 1 - cloud runner image build and GHCR push
FR4: Epic 1 - self-hosted runner executes internal release steps
FR5: Epic 2 - manifest promotion into `terminus.infra` values files
FR6: Epic 3 - Temporal start with extended release input
FR7: Epic 3 - environment-aware workflow ID scheme
FR8: Epic 2 - two-environment deployment model in one cluster
FR9: Epic 2 - dev ArgoCD, values, ingress, DNS assets
FR10: Epic 3 - preserve existing release contract activities
FR11: Epic 3 - environment-aware routing for release activities
FR12: Epic 4 - Semaphore remains break-glass fallback only
FR13: Epic 4 - shared self-hosted k3s Actions runner
FR14: Epic 2 - promote the same artifact from dev to prod without rebuild

## Epic List

### Epic 1: Reliable Release Entry Path
Developers can merge a PR and have the platform automatically build, tag, and hand off a release through the correct execution path without manual intervention.
**FRs covered:** FR1, FR2, FR3, FR4

### Epic 2: GitOps Environment Promotion
Operators and developers can see a release promoted into the correct environment through auditable manifest updates, with dev and prod kept distinct but consistent.
**FRs covered:** FR5, FR8, FR9, FR14

### Epic 3: Environment-Aware Release Orchestration
The platform can start and execute the existing Temporal release contract with the right environment context, activity routing, and workflow identity for each release.
**FRs covered:** FR6, FR7, FR10, FR11

### Epic 4: Operational Safety Net and Runner Infrastructure
The platform can execute internal-network release steps through a shared k3s runner, preserve a controlled break-glass path, and keep credentials and operations inside the governed platform model.
**FRs covered:** FR12, FR13

---

## Epic 1: Reliable Release Entry Path

Developers can merge a PR and have the platform automatically build, tag, and hand off a release through the correct execution path without manual intervention.

**Goal:** Make git merge the only normal approval signal and establish a deterministic GitHub Actions entry path that produces a release-ready artifact and hands control to internal execution safely.

### Story 1.1: Merge-Driven Release Workflow Trigger

As a service developer,
I want merged PRs to `develop` and `main` to trigger the release workflow automatically,
So that release approval is expressed only through git merge and not through a separate operator action.

**Acceptance Criteria:**

**Given** a PR is closed on any branch
**When** the GitHub Actions release workflow evaluates the event
**Then** it runs only when `github.event.pull_request.merged == true`
**And** it ignores unmerged PR closures and non-release target branches.

**Given** a merged PR targets `develop` or `main`
**When** the release workflow starts
**Then** it resolves the target environment as `dev` for `develop` and `prod` for `main`
**And** it exposes the resolved environment and service identity as outputs for downstream jobs.

**Given** the release workflow is the normal path
**When** the workflow documentation and job summary are generated
**Then** they state that PR merge is the release approval
**And** they do not instruct operators to use Semaphore for routine releases.

**References:** FR1, FR2, NFR1, NFR2, NFR10

### Story 1.2: Cloud Runner Build and Immutable Image Publication

As a service developer,
I want the workflow to build and publish an immutable SHA-tagged image on the cloud runner path,
So that each release candidate is traceable to a specific commit and suitable for promotion.

**Acceptance Criteria:**

**Given** a merged PR targets `develop`
**When** the cloud runner build job executes
**Then** it builds the service image and pushes `ghcr.io/{org}/{service}:{sha}`
**And** it records the full pushed tag as an output for downstream jobs.

**Given** the build job publishes an image
**When** tagging rules are applied
**Then** the immutable SHA tag is the release artifact of record
**And** the normal release path does not depend on `latest`.

**Given** the image push fails
**When** the build job completes
**Then** the workflow fails in the cloud runner stage
**And** no internal release step is started.

**References:** FR3, NFR2, NFR3, NFR8

### Story 1.3: Self-Hosted Runner Release Handoff

As a platform operator,
I want internal release steps to execute on a self-hosted k3s runner,
So that manifest promotion and Temporal invocation can reach `*.trantor.internal` services safely.

**Acceptance Criteria:**

**Given** the release workflow needs to talk to internal platform services
**When** the post-build release job is declared
**Then** it runs on `[self-hosted, trantor-internal]`
**And** it does not run on a GitHub-hosted cloud runner.

**Given** the build job completed successfully
**When** the self-hosted release job starts
**Then** it receives the resolved environment, service, and image tag outputs from prior jobs
**And** it uses those outputs as the single source of truth for downstream promotion logic.

**Given** the self-hosted runner is unavailable or mislabeled
**When** the workflow scheduler cannot place the internal release job
**Then** the workflow remains failed and visible in GitHub Actions
**And** no operator-facing instruction suggests silently bypassing the normal path.

**References:** FR4, NFR6, NFR8, NFR10

### Story 1.4: Release Gating and Audit Summary

As a platform operator,
I want the workflow to stop cleanly before release execution when prerequisites fail,
So that the automated path remains auditable and does not create unsafe partial releases.

**Acceptance Criteria:**

**Given** any prerequisite stage in the normal release path fails
**When** the workflow evaluates downstream job conditions
**Then** manifest promotion and Temporal start are skipped
**And** the workflow summary records the failed stage, branch, environment, and image context.

**Given** a release workflow completes or fails
**When** an operator reviews the GitHub Actions run
**Then** the run summary clearly distinguishes build failure, promotion failure, and Temporal start failure
**And** it preserves an auditable trail for the exact release attempt.

**Given** the normal path failed before Temporal start
**When** operators need to recover
**Then** the audit output points to the documented break-glass path as exceptional follow-up
**And** it does not redefine Semaphore as the primary entry point.

**References:** FR1, FR4, NFR2, NFR8, NFR10

---

## Epic 2: GitOps Environment Promotion

Operators and developers can see a release promoted into the correct environment through auditable manifest updates, with dev and prod kept distinct but consistent.

**Goal:** Establish a single environment contract across GitHub Actions, `terminus.infra`, ArgoCD, ingress, and DNS, then make release promotion update those assets safely and repeatably.

### Story 2.1: Canonical Environment Mapping Contract

As a platform maintainer,
I want a single canonical mapping for branch, environment, namespace, values file, ArgoCD app, and smoke-test naming,
So that release automation and infra assets do not drift across services.

**Acceptance Criteria:**

**Given** the release design spans multiple repos and systems
**When** the environment contract is implemented
**Then** one authoritative mapping defines `develop -> dev` and `main -> prod`
**And** it also defines namespace, values file, ArgoCD app, ingress host, and smoke-test naming for each environment.

**Given** downstream workflow and infra stories consume environment metadata
**When** they read environment-specific names
**Then** they source them from the canonical mapping
**And** they do not duplicate independent naming logic in multiple places.

**Given** a new service adopts deliverycontrol
**When** its release assets are created
**Then** the service follows the same environment mapping contract without inventing local exceptions.

**References:** FR8, FR9, NFR12

### Story 2.2: Dev GitOps Assets for a Service

As a service developer,
I want dev values and ArgoCD applications created alongside prod assets,
So that merged code can be promoted into a distinct dev environment without disturbing production.

**Acceptance Criteria:**

**Given** a service currently has only prod deployment assets
**When** dev GitOps assets are added in `terminus.infra`
**Then** `values-dev.yaml` exists for the service with dev-specific image override behavior
**And** the service has dev ArgoCD application definitions that target the `develop` branch and `{service}-dev` namespace.

**Given** the dev application set is created
**When** ArgoCD renders the service manifests
**Then** the dev app uses the dev values file and dev namespace
**And** the prod app remains on its production branch and namespace.

**Given** a service has no existing dev assets
**When** the story is complete
**Then** the service can be synced into dev independently of prod
**And** the naming matches the canonical environment mapping.

**References:** FR8, FR9, NFR4, NFR12

### Story 2.3: Environment-Specific Ingress, DNS, and Smoke-Test Assets

As a platform operator,
I want each environment to have its own ingress, DNS, and smoke-test path,
So that release verification targets the correct live endpoint for dev and prod.

**Acceptance Criteria:**

**Given** a service participates in deliverycontrol
**When** environment-specific networking assets are created
**Then** dev ingress uses `{service}-dev.trantor.internal` and namespace `{service}-dev`
**And** prod ingress remains `{service}.trantor.internal` in namespace `{service}`.

**Given** the service requires DNS resolution for smoke tests
**When** the operator-facing infra assets are updated
**Then** the required dev DNS record and supporting operational notes exist
**And** the verification path uses external DNS hosts rather than cluster-internal service addresses.

**Given** Semaphore smoke tests are part of the release contract
**When** environment-specific verify templates are registered
**Then** both `verify-{service}-dev-deploy` and `verify-{service}-deploy` exist where required
**And** each targets the environment-specific host.

**References:** FR9, FR11, NFR6, NFR12

### Story 2.4: Safe Manifest Promotion for Dev Releases

As a platform operator,
I want dev releases to update `terminus.infra` through a controlled commit on the self-hosted runner,
So that GitOps promotion is auditable, revertible, and safe under concurrent merge pressure.

**Acceptance Criteria:**

**Given** a merged PR to `develop` produced a SHA-tagged image
**When** the self-hosted release job promotes the release
**Then** it updates only the dev values target for that service in `terminus.infra`
**And** it creates a commit using the approved release commit message convention.

**Given** multiple merges may land close together
**When** the promotion logic pushes to `terminus.infra`
**Then** it uses explicit serialization or rebase-and-retry behavior to avoid unordered or conflicting writes
**And** promotion failure remains visible to the workflow as a hard failure.

**Given** manifest promotion fails
**When** the self-hosted job exits
**Then** it does not invoke Temporal
**And** the failed promotion remains identifiable as the stopping point.

**References:** FR5, NFR2, NFR4, NFR8, NFR11

### Story 2.5: Production Promotion Reuses the Dev-Validated Artifact

As a service developer,
I want the production promotion path to reuse the artifact that already passed dev,
So that production deploys exactly the image that was validated earlier and does not rebuild a different binary.

**Acceptance Criteria:**

**Given** a merged PR targets `main`
**When** the production promotion path runs
**Then** it resolves the previously built SHA artifact for the merged code
**And** it does not rebuild a new image for the production release.

**Given** the production promotion job updates `terminus.infra`
**When** it writes the prod values target
**Then** it commits the same SHA that was validated in dev
**And** the commit remains traceable to that original artifact.

**Given** the required artifact cannot be resolved or does not exist in GHCR
**When** the production promotion path evaluates the release
**Then** the workflow fails before manifest promotion or Temporal start
**And** the error makes clear that artifact reuse was the violated precondition.

**References:** FR5, FR14, NFR3, NFR9

---

## Epic 3: Environment-Aware Release Orchestration

The platform can start and execute the existing Temporal release contract with the right environment context, activity routing, and workflow identity for each release.

**Goal:** Preserve the current release orchestration model while extending it to be environment-aware, backwards compatible, and correctly invoked from the new GitHub Actions release path.

### Story 3.1: Backward-Compatible ReleaseInput Extension

As a platform developer,
I want `ReleaseInput` extended with environment-aware fields,
So that the existing Temporal release contract can distinguish dev and prod while still accepting legacy manual starts.

**Acceptance Criteria:**

**Given** `ReleaseWorkflow` already accepts release input
**When** the input type is extended
**Then** it includes `Environment` and `ImageTag` in addition to existing fields
**And** older manual invocations that omit those fields remain compatible with a safe default behavior.

**Given** the input type changes
**When** tests are added for serialization and defaulting
**Then** both legacy and new payload shapes are covered
**And** the workflow can distinguish explicit `dev` and `prod` execution contexts.

**Given** the input type is used operationally
**When** operators inspect workflow history or logs
**Then** the environment and image tag are visible for audit purposes
**And** secrets are not introduced into workflow input.

**References:** FR6, NFR2, NFR5, NFR7

### Story 3.2: Environment-Aware Workflow Identity and Activity Routing

As a platform developer,
I want workflow identity and activity routing to include environment context,
So that dev and prod releases for the same SHA remain distinct and execute the correct verification path.

**Acceptance Criteria:**

**Given** the same SHA can be promoted to dev and later to prod
**When** workflow IDs are created
**Then** they follow `release-{service}-{env}-{sha[:8]}`
**And** the ID policy prevents accidental duplicate successful releases while permitting reruns after failure.

**Given** a release runs in `dev` or `prod`
**When** activity templates are resolved
**Then** smoke-test activity routing selects `verify-{service}-dev-deploy` for dev and `verify-{service}-deploy` for prod
**And** secret seeding remains compatible with the shared seeding convention.

**Given** environment-aware routing is implemented
**When** automated tests execute
**Then** they cover dev and prod template selection, workflow ID formatting, and optional database behavior
**And** they verify no regression to the original release contract.

**References:** FR7, FR10, FR11, NFR5, NFR12

### Story 3.3: Temporal Start After Confirmed Manifest Promotion

As a platform operator,
I want GitHub Actions to start `ReleaseWorkflow` only after manifest promotion is confirmed pushed,
So that Temporal always observes a release candidate with a corresponding GitOps commit to wait on.

**Acceptance Criteria:**

**Given** the self-hosted release job completed manifest promotion successfully
**When** it starts Temporal
**Then** it invokes `ReleaseWorkflow` with `ServiceName`, `ProvisionDB`, `Environment`, and `ImageTag`
**And** it uses the environment-aware workflow ID and duplicate-failed-only reuse policy.

**Given** the Temporal start path uses cluster-local tools
**When** the invocation is implemented
**Then** it runs from the self-hosted runner through the approved internal access method
**And** it does not depend on cloud-runner network reachability.

**Given** `temporal workflow start` fails after a successful promotion commit
**When** the job ends
**Then** the GitHub Actions run is marked failed and identifies Temporal start as the failure boundary
**And** operators can correlate the failure to the specific promoted commit and workflow ID.

**References:** FR6, FR10, NFR2, NFR6, NFR8

### Story 3.4: Preserved Release Contract Sequencing and Observability

As a platform operator,
I want the new release path to preserve the established release activity sequence while improving observability,
So that automated releases remain understandable and debuggable without changing the core orchestration contract.

**Acceptance Criteria:**

**Given** the release is started from GitHub Actions instead of manually from Semaphore
**When** `ReleaseWorkflow` executes
**Then** it still sequences `SeedSecrets`, optional `ProvisionDatabase`, `WaitForArgoCDSync`, and `RunSmokeTest`
**And** the orchestration semantics remain consistent with the existing architecture.

**Given** a release is in progress or has completed
**When** operators review GitHub Actions, Temporal, or runbook guidance
**Then** they can identify the release by branch, environment, SHA, and workflow ID
**And** the observability path makes GitHub Actions visibly primary and Semaphore visibly fallback only.

**Given** a release fails in an activity stage
**When** operators investigate
**Then** the failure boundary is attributable to the specific contract stage
**And** the documentation remains aligned with the implemented sequence.

**References:** FR10, FR11, NFR2, NFR5, NFR10

### Story 3.5: Temporal-Controlled ArgoCD Sync After Release Prerequisites

As a platform operator,
I want `ReleaseWorkflow` to trigger ArgoCD sync only after secrets and database prerequisites are complete,
So that release-managed apps do not begin deployment before Temporal has finished the steps that make the deployment safe to start.

**Acceptance Criteria:**

**Given** manifest promotion already committed the desired image tag and `ReleaseWorkflow` has started
**When** `SeedSecrets` and optional `ProvisionDatabase` complete successfully
**Then** Temporal explicitly requests sync for the environment-specific ArgoCD application
**And** `WaitForArgoCDSync` runs only after that explicit sync step.

**Given** a release-managed app is participating in this orchestration model
**When** GitOps state changes before Temporal reaches the sync step
**Then** ArgoCD auto-deploy does not roll the workload forward ahead of Temporal's sync decision
**And** the release design still preserves Git as the source of desired state.

**Given** the explicit ArgoCD sync request fails or is rejected
**When** the workflow handles that failure
**Then** the release stops before smoke testing
**And** operators can identify the failure as a distinct sync-trigger boundary in workflow output and logs.

**References:** FR10, FR11, NFR2, NFR5, NFR8

---

## Epic 4: Operational Safety Net and Runner Infrastructure

The platform can execute internal-network release steps through a shared k3s runner, preserve a controlled break-glass path, and keep credentials and operations inside the governed platform model.

**Goal:** Provide the secure infrastructure and fallback mechanisms that make the new release path operable in the homelab without letting fallback behavior become the default release system.

### Story 4.1: Shared k3s GitHub Actions Runner Deployment

As a platform operator,
I want a shared self-hosted GitHub Actions runner deployed in k3s,
So that internal release jobs have a governed execution environment with cluster-reachable network access.

**Acceptance Criteria:**

**Given** release jobs must reach internal services
**When** the runner deployment is added to `terminus.infra`
**Then** it includes deployment, RBAC, and secret delivery assets needed for registration and execution
**And** it advertises the labels required by the release workflow.

**Given** runner registration requires credentials
**When** runner secrets are provisioned
**Then** they follow the Vault KV to ESO to k8s Secret pattern
**And** no static registration token is committed to git.

**Given** the runner is deployed by ArgoCD
**When** operators inspect the cluster state
**Then** the runner appears as a managed workload in k3s
**And** its purpose as the internal release execution path is documented.

**References:** FR13, NFR6, NFR7

### Story 4.2: Runner Git Access and Promotion Permissions

As a platform operator,
I want the self-hosted runner to have the minimum git access needed to promote manifests,
So that it can push controlled release commits to `terminus.infra` without broad unmanaged repository privilege.

**Acceptance Criteria:**

**Given** the self-hosted runner must push release commits
**When** repository credentials are configured
**Then** they are delivered through approved secret management paths
**And** they are scoped to the repositories and actions needed for release promotion.

**Given** the runner performs a promotion commit
**When** the push succeeds
**Then** the resulting git identity is attributable to the automation path
**And** the audit trail shows that the commit originated from controlled release automation.

**Given** repository access is missing or insufficient
**When** the promotion job attempts to push
**Then** the workflow fails before Temporal start
**And** the missing permission is surfaced as an operational setup defect.

**References:** FR5, FR13, NFR2, NFR7, NFR8

### Story 4.3: Break-Glass Release Path and Operator Guardrails

As a platform operator,
I want a documented manual release fallback that mirrors the normal release contract,
So that releases can proceed during GitHub Actions outages without normalizing manual invocation.

**Acceptance Criteria:**

**Given** GitHub Actions is unavailable or unusable for a release
**When** an operator invokes the break-glass path
**Then** Semaphore can start the same `ReleaseWorkflow` contract with the environment-aware input shape
**And** the workflow still runs through the same downstream release stages.

**Given** the break-glass path exists
**When** operators read the runbook and supporting docs
**Then** they are told it is exceptional and not the default release path
**And** the normal GitHub Actions route remains the first documented procedure.

**Given** a manual fallback release is used
**When** the operator records or inspects the run
**Then** the environment, SHA, and workflow ID remain traceable
**And** the fallback does not invent a separate incompatible release contract.

### Story 4.4: Automatic Semaphore Template Reconciliation On Merge

As a platform operator,
I want Semaphore project configuration to reconcile automatically after `terminus.infra` merges,
So that onboarding a new release-managed service does not require a manual OpenTofu apply before Temporal can trigger the expected Semaphore tasks.

**Acceptance Criteria:**

**Given** `terminus.infra` merges a change that modifies the Semaphore OpenTofu environment
**When** the protected branch update lands on the automation path
**Then** a GitHub Actions workflow on the internal self-hosted runner runs `tofu init`, `tofu plan`, and `tofu apply` for `tofu/environments/semaphore`
**And** the outcome is visible in GitHub Actions without requiring an operator to log into the runner manually.

**Given** the Semaphore OpenTofu environment is now part of unattended automation
**When** the workflow applies it repeatedly across merges
**Then** it uses a durable remote state backend instead of ephemeral runner-local state
**And** concurrent reconcile runs serialize safely.

**Given** the workflow needs live credentials for Vault, Consul, and the Semaphore API
**When** it executes on the internal runner
**Then** it sources those credentials from approved internal automation paths rather than committing them to git
**And** a missing prerequisite fails the workflow before partial configuration drift is introduced.

**Given** a service merge depends on newly declared Semaphore templates
**When** that template declaration reaches `main`
**Then** the templates are made live by the normal automation path before the next runtime release depends on them
**And** the release model continues to satisfy the requirement that PR merge triggers the entire normal process.

**References:** FR13, NFR1, NFR2, NFR7, NFR8

**References:** FR12, NFR2, NFR5, NFR10