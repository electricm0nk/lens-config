---
phase: devproposal
initiative: terminus-platform-deliverycontrol
generatedFrom: epics.md
date: 2026-04-11
---

# terminus-platform-deliverycontrol — Story Index

All stories are fully defined in [epics.md](epics.md) with complete Given/When/Then acceptance criteria. This index provides the sprint-planning summary view.

---

## Epic 1: Reliable Release Entry Path

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 1.1 | Merge-Driven Release Workflow Trigger | FR1, FR2 | `release.yml` runs only on merged PRs to `develop` and `main`; branch resolves to environment outputs |
| 1.2 | Cloud Runner Build and Immutable Image Publication | FR3 | Cloud runner builds and pushes `ghcr.io/{org}/{service}:{sha}`; immutable tag recorded for downstream jobs |
| 1.3 | Self-Hosted Runner Release Handoff | FR4 | Internal release job runs on `[self-hosted, trantor-internal]` with build outputs passed forward |
| 1.4 | Release Gating and Audit Summary | FR1, FR4 | Downstream jobs gated on prior success; GitHub Actions summary distinguishes failure boundaries |

---

## Epic 2: GitOps Environment Promotion

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 2.1 | Canonical Environment Mapping Contract | FR8, FR9 | One authoritative mapping for branch, environment, namespace, values file, app, ingress, and smoke-test names |
| 2.2 | Dev GitOps Assets for a Service | FR8, FR9 | `values-dev.yaml` plus dev ArgoCD apps targeting `develop` and `{service}-dev` |
| 2.3 | Environment-Specific Ingress, DNS, and Smoke-Test Assets | FR9, FR11 | Dev/prod ingress hosts, DNS expectations, and verify templates are explicitly separated |
| 2.4 | Safe Manifest Promotion for Dev Releases | FR5 | Self-hosted runner commits dev image tag to `terminus.infra` with serialization or retry protection |
| 2.5 | Production Promotion Reuses the Dev-Validated Artifact | FR5, FR14 | `main` promotion resolves and reuses the previously validated SHA instead of rebuilding |

---

## Epic 3: Environment-Aware Release Orchestration

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 3.1 | Backward-Compatible ReleaseInput Extension | FR6 | `ReleaseInput` adds `Environment` and `ImageTag` without breaking legacy manual starts |
| 3.2 | Environment-Aware Workflow Identity and Activity Routing | FR7, FR10, FR11 | Workflow IDs and template routing include environment context |
| 3.3 | Temporal Start After Confirmed Manifest Promotion | FR6, FR10 | GitHub Actions starts `ReleaseWorkflow` only after manifest push is confirmed |
| 3.4 | Preserved Release Contract Sequencing and Observability | FR10, FR11 | Existing activity order preserved with clearer GitHub Actions and Temporal observability |
| 3.5 | Temporal-Controlled ArgoCD Sync After Release Prerequisites | FR10, FR11 | Temporal explicitly triggers env-specific ArgoCD sync after secrets and optional DB work complete |

---

## Epic 4: Operational Safety Net and Runner Infrastructure

| Story | Title | FRs | Key Deliverable |
|---|---|---|---|
| 4.1 | Shared k3s GitHub Actions Runner Deployment | FR13 | ArgoCD-managed runner deployment with Vault/ESO-delivered registration credentials |
| 4.2 | Runner Git Access and Promotion Permissions | FR5, FR13 | Narrowly scoped automation credentials for `terminus.infra` promotion commits |
| 4.3 | Break-Glass Release Path and Operator Guardrails | FR12 | Semaphore fallback mirrors normal release contract and remains explicitly exceptional |
| 4.4 | Automatic Semaphore Template Reconciliation On Merge | FR13 | `terminus.infra` auto-applies Semaphore project config after merge so new release templates become live without operator follow-up |

---

## Summary

| Metric | Count |
|---|---|
| Epics | 4 |
| Stories | 18 |
| FRs covered | 14/14 |
| NFRs covered | 12/12 |
| Readiness verdict | READY_WITH_NOTES |
| Primary target repo | `terminus.platform` |