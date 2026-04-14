# Implementation Readiness Checklist — terminus-platform-deliverycontrol (DevProposal)

**Date:** 2026-04-11
**Initiative:** terminus-platform-deliverycontrol
**Audience:** medium (lead review)
**Phase:** devproposal
**Mode:** validate
**Reviewer:** Lord Commander Creed (PM)

---

## Document Inventory

| Artifact | Path | Status |
|---|---|---|
| Architecture | `docs/terminus/platform/deliverycontrol/architecture.md` | Present — release contract and implementation decisions locked |
| Adversarial Review | `docs/terminus/platform/deliverycontrol/adversarial-review-report.md` | PASS_WITH_NOTES |
| Epics & Stories | `docs/terminus/platform/deliverycontrol/epics.md` | Present — 4 epics, 17 stories |
| Story Index | `docs/terminus/platform/deliverycontrol/stories.md` | Present — stakeholder-readable story summary |

---

## FR Coverage Validation

All 14 functional requirements are covered by the approved stories.

| FR | Story Coverage | Description |
|---|---|---|
| FR1 | 1.1, 1.4 | PR merge is the only normal release approval signal |
| FR2 | 1.1 | `release.yml` triggers on merged PRs to `develop` and `main` |
| FR3 | 1.2 | Cloud runner builds and pushes immutable SHA-tagged image |
| FR4 | 1.3, 1.4 | Internal release steps run on self-hosted k3s runner |
| FR5 | 2.4, 2.5, 4.2 | Manifest promotion commits image tag into `terminus.infra` |
| FR6 | 3.1, 3.3 | Temporal starts with `ServiceName`, `ProvisionDB`, `Environment`, `ImageTag` |
| FR7 | 3.2, 3.3 | Workflow IDs use `release-{service}-{env}-{sha[:8]}` |
| FR8 | 2.1, 2.2 | Dev and prod environments exist in one cluster |
| FR9 | 2.1, 2.2, 2.3 | Dev ArgoCD, values, ingress, and DNS assets exist |
| FR10 | 3.2, 3.3, 3.4, 3.5 | Existing release contract sequence is preserved |
| FR11 | 2.3, 3.2, 3.4, 3.5 | Activity routing and verification are environment-aware |
| FR12 | 4.3 | Semaphore remains the break-glass release path |
| FR13 | 4.1, 4.2 | Shared self-hosted k3s runner is deployed with governed credentials |
| FR14 | 2.5 | Production promotion reuses the dev-validated artifact |

**FR Coverage: 14/14 — COMPLETE**

---

## NFR Coverage Validation

| NFR | Story Coverage | Verification Intent |
|---|---|---|
| NFR1 | 1.1 | No human action after merge on normal path |
| NFR2 | 1.1, 1.4, 2.4, 3.4, 3.5, 4.2, 4.3 | Audit trail across git, Actions, and Temporal |
| NFR3 | 1.2, 2.5 | SHA tags only; no `latest` dependency |
| NFR4 | 2.2, 2.4 | Manifest promotion remains revertible via single commit |
| NFR5 | 3.1, 3.2, 3.4, 3.5, 4.3 | Backward compatibility with existing release workflow and manual fallback |
| NFR6 | 1.3, 2.3, 3.3, 4.1 | Internal-network steps run only on self-hosted runner |
| NFR7 | 3.1, 4.1, 4.2 | Vault → ESO → k8s Secret credential handling |
| NFR8 | 1.2, 1.4, 2.4, 3.3, 3.5, 4.2 | Fail fast on promotion or credential failures |
| NFR9 | 2.5 | Prod uses same artifact already validated in dev |
| NFR10 | 1.1, 1.4, 3.4, 4.3 | Break-glass remains secondary |
| NFR11 | 2.4 | Concurrent manifest promotion handled safely |
| NFR12 | 2.1, 2.2, 2.3, 3.2 | Canonical environment mapping prevents naming drift |

**NFR Coverage: 12/12 — COMPLETE**

---

## Architecture Compliance

| Constraint | Status |
|---|---|
| GitHub Actions is the primary release entry point | Complete |
| Temporal remains the release orchestrator | Complete |
| ArgoCD remains the only delivery mechanism | Complete |
| Semaphore is break-glass only | Complete |
| `ReleaseInput` extends backward-compatibly with `Environment` and `ImageTag` | Complete |
| Dev and prod naming follow one canonical environment contract | Complete |
| `develop -> dev`, `main -> prod` contract preserved | Complete |
| Self-hosted runner required for `*.trantor.internal` access | Complete |
| Manifest promotion precedes Temporal start | Complete |
| `develop -> main` reuses previously validated artifact | Complete |

---

## Story Quality Assessment

| Check | Result |
|---|---|
| All stories use user-story format | Pass |
| All stories include Given/When/Then acceptance criteria | Pass |
| Stories are single-agent completable | Pass |
| No forward dependencies inside any epic | Pass |
| Epic order supports incremental delivery | Pass |
| Break-glass path does not displace normal path | Pass |

---

## Readiness Verdict

**READY FOR SPRINT PLANNING**

- 14/14 FRs covered
- 12/12 NFRs covered
- 4 epics, 17 stories
- No blocker found in planning artifacts
- Remaining notes are execution choices, not planning gaps