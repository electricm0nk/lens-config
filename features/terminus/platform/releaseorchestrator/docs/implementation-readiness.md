---
verdict: READY
stepsCompleted: [1, 2, 3, 4, 5, 6]
mode: adversarial
assessedAt: '2026-04-10'
project_name: terminus-platform-releaseorchestrator
assessor: check-implementation-readiness (adversarial, batch)
inputDocuments:
  - docs/terminus/platform/releaseorchestrator/architecture.md
  - docs/terminus/platform/releaseorchestrator/adversarial-review-report.md
  - docs/terminus/platform/releaseorchestrator/epics.md
initiative_track: tech-change
---

# Implementation Readiness Assessment — terminus-platform-releaseorchestrator

**Date:** 2026-04-10
**Project:** terminus-platform-releaseorchestrator
**Mode:** Adversarial (batch)
**Track:** tech-change (no separate PRD — architecture.md is the requirements source)

---

## Document Inventory

| Document | Path | Status |
|---|---|---|
| Architecture | `docs/terminus/platform/releaseorchestrator/architecture.md` | ✅ Complete (`stepsCompleted: [1-8]`) |
| Adversarial Review | `docs/terminus/platform/releaseorchestrator/adversarial-review-report.md` | ✅ Complete (`verdict: PASS_WITH_NOTES`) |
| Epics & Stories | `docs/terminus/platform/releaseorchestrator/epics.md` | ✅ Complete (`stepsCompleted: [1,2,3,4]`) |
| PRD | N/A | ℹ️ Not required — `tech-change` track uses architecture as requirements source |
| UX Design | N/A | ℹ️ Not required — no user-facing interface changes |

---

## Requirements Coverage Analysis

Architecture defines 12 FRs and 10 NFRs. Coverage cross-referenced against epics.md stories.

### Functional Requirements

| FR | Requirement | Epic | Story | Status |
|---|---|---|---|---|
| FR1 | `ReleaseWorkflow` sequences SeedSecrets → [ProvisionDB] → WaitForArgoCDSync → RunSmokeTest | 2 | 2.2 | ✅ Covered |
| FR2 | `SeedSecrets` — Semaphore REST API fire-and-poll | 2 | 2.3 | ✅ Covered |
| FR3 | `ProvisionDatabase` — Semaphore fire-and-poll; skip when `ProvisionDB: false` | 2 | 2.4 | ✅ Covered |
| FR4 | `WaitForArgoCDSync` — poll every 15s with `RecordHeartbeat`; fail after 10min | 2 | 2.5 | ✅ Covered |
| FR5 | `RunSmokeTest` — Semaphore REST API fire-and-poll | 2 | 2.6 | ✅ Covered |
| FR6 | Accept `ReleaseInput` on `terminus-platform` task queue | 2 | 2.1, 2.2 | ✅ Covered |
| FR7 | Workflow ID pattern `release-{service}-{sha[:8]}`; `ALLOW_DUPLICATE_FAILED_ONLY` | 2 | 2.2 | ✅ Covered |
| FR8 | Migrate `DailyBriefingWorkflow` TypeScript stub → Go stub | 1 | 1.2 | ✅ Covered |
| FR9 | Replace Node.js Dockerfile with Go multi-stage Dockerfile | 1 | 1.3 | ✅ Covered |
| FR10 | Remove all TypeScript scaffold files after Go binary compiles + CI updated | 1 | 1.4 | ✅ Covered |
| FR11 | Declare `ReleaseInput`, `ReleaseResult`, `ModulePayload[S any]` in `internal/types/` | 1 | 1.1 | ✅ Covered |
| FR12 | Two `ExternalSecret` manifests: `release-semaphore-token`, `release-argocd-token` | 3 | 3.1 | ✅ Covered |

**FR Coverage: 12/12 — No gaps.**

### Non-Functional Requirements

| NFR | Requirement | Story Coverage | Status |
|---|---|---|---|
| NFR1 | Determinism — no `time.Now()`, `os.Getenv`, `http` in workflow functions | 2.2 AC explicit | ✅ |
| NFR2 | Credentials via Vault KV → ESO → k8s Secret → env var | 2.3–2.6, 3.1 pre-conditions | ✅ |
| NFR3 | `slog.Info`/`slog.Error` with structured KV pairs; no `fmt.Println` | 2.3–2.6 AC explicit | ✅ |
| NFR4 | `activity.RecordHeartbeat` on every poll cycle in `WaitForArgoCDSync` | 2.5 AC explicit | ✅ |
| NFR5 | 80% line coverage minimum; 100% on type validation | 1.2, 2.6 AC | ✅ |
| NFR6 | TDD — `_test.go` scaffolded before implementation | All stories require `_test.go` | ✅ |
| NFR7 | `os.Getenv` in `main.go` or activity constructors only | 2.3–2.6 AC explicit | ✅ |
| NFR8 | No `interface{}` in workflow/activity signatures | 2.1 AC explicit | ✅ |
| NFR9 | `temporal.NewNonRetryableApplicationError` for unrecoverable errors | 2.3–2.6 AC explicit | ✅ |
| NFR10 | Go 1.25; module `github.com/electricm0nk/terminus-platform` | 1.1 AC explicit | ✅ |

**NFR Coverage: 10/10 — No gaps.**

### Adversarial Review Constraint Compliance

| Finding | Constraint | Story | Status |
|---|---|---|---|
| Finding 2 | Vault seeding pre-condition in all activity stories | 2.3, 2.4, 2.5, 2.6, 3.1 | ✅ Pre-condition blocks included |
| Finding 3 | TypeScript removal MUST NOT merge before CI updated | Story 1.4 explicit pre-condition | ✅ Ordering enforced |

---

## Epic Quality Review (Adversarial)

### Epic Structure and Independence

**Note on tech-change track:** This is a `tech-change` initiative. Technical epics are the appropriate epic type — user-value framing is secondary; migration safety and structural integrity are primary quality criteria.

**Epic 1: Go Worker Foundation**
- Independence: Fully self-contained. Delivers a compilable, deployable worker — independently valuable.
- Story sequence: 1.1 → 1.2 → 1.3 → 1.4 is correctly ordered. Each story unlocks the next.
- No forward dependencies detected.
- Verdict: ✅ Sound

**Epic 2: Release Workflow**
- Independence: Depends on Epic 1 (stated explicitly). Internally, stories 2.1 → 2.2 → 2.3–2.6 are correctly ordered.
- Stories 2.3–2.6 are sequential but could theoretically be parallelised (different activity functions). Sequential ordering is conservative and safe.
- Verdict: ✅ Sound

**Epic 3: Credential Infrastructure**
- Independence: Story 3.1 can be executed in parallel with Epic 2 — explicitly called out in dependency order.
- Single story; well-scoped.
- Verdict: ✅ Sound

---

## Adversarial Findings

### Finding IR-1 — DailyBriefingSchema not declared in Story 1.1 types *(NOTE)*

**Category:** Story completeness  
**Severity:** NOTE  
**Story:** 1.2

**Observation:** Story 1.2 references `types.ModulePayload[DailyBriefingSchema]` as the workflow input type, but `DailyBriefingSchema` is not declared in Story 1.1 (which covers `internal/types/`). The type needs to live somewhere — either in `internal/types/daily_briefing.go` or as a local type in `internal/workflows/daily_briefing.go`.

**Recommendation:** When implementing Story 1.2, define `DailyBriefingSchema` as a local struct in `internal/workflows/daily_briefing.go` (it's workflow-specific state, not a shared domain type). No story change required.

**Resolution path:** Implementor discretion. DOES NOT block proceeding.

---

### Finding IR-2 — ReleaseResult.SmokeTestPassed assembly is implicit *(NOTE)*

**Category:** Story completeness  
**Severity:** NOTE  
**Stories:** 2.2, 2.6

**Observation:** Story 2.6 mentions `"sets ReleaseResult.SmokeTestPassed = true — propagated by workflow"` in parentheses, implying the workflow assembles this from `RunSmokeTest` success. However, `RunSmokeTest` returns `error` (not a typed result with a boolean), so the workflow infers "smoke test passed" from `error == nil`. Story 2.2 does not have explicit AC requiring the workflow to set `ReleaseResult.SmokeTestPassed = true` on `RunSmokeTest` success.

**Recommendation:** During Story 2.2 or 2.6 implementation, the workflow should assemble `ReleaseResult{SmokeTestPassed: true}` when `RunSmokeTest` returns nil. This is straightforward but should not be forgotten. No story change required.

**Resolution path:** Implementor awareness. DOES NOT block proceeding.

---

### Finding IR-3 — Activity constructor injection pattern not specified *(NOTE)*

**Category:** Story completeness  
**Severity:** NOTE  
**Stories:** 2.3, 2.4, 2.5, 2.6

**Observation:** Stories 2.3–2.6 specify `os.Getenv` via "constructor injection into the activity — not `os.Getenv` inside the function body" (per NFR7). However, no story defines the constructor struct pattern — i.e., whether activities are methods on a struct like:
```go
type ReleaseActivities struct {
    SemaphoreAPIToken  string
    SemaphoreProjectID string
    ArgoCDAPIToken     string
    ArgoCDBaseURL      string
}
```
...or a different injection approach. The pattern needs consistency across all four release activities to avoid re-registering incompatible types with the Temporal worker.

**Recommendation:** When implementing Story 2.1 (stubs), define `ReleaseActivities` struct at the top of `internal/activities/release.go` and make all release activity functions methods on this struct. This struct is populated in `cmd/worker/main.go` from `os.Getenv` calls. Worker registration then uses an instance of this struct.

**Resolution path:** Implementor awareness for Story 2.1. DOES NOT block proceeding.

---

### Finding IR-4 — Story 3.1 references SecretStore with no terminus.platform ESO precedent *(NOTE)*

**Category:** Story completeness  
**Severity:** NOTE  
**Story:** 3.1

**Observation:** Story 3.1 AC states: "both manifests follow the same `SecretStore` reference and `refreshInterval` pattern as existing ESO manifests in the repository." However, the `terminus.platform` repository currently has no existing ESO manifests (it's a TypeScript worker with no k8s credentials). The developer will need to look up the `SecretStore` name from either `terminus.infra` or the `fourdogs-central` k8s manifests.

**Recommendation:** When implementing Story 3.1, reference the `ClusterSecretStore` name from an existing working ESO manifest — e.g., from `fourdogs-central/k8s/` or `terminus.infra/k8s/eso/`. This is a lookup, not a design decision. DOES NOT block proceeding.

**Resolution path:** Implementor lookup required. DOES NOT block proceeding.

---

## Summary and Recommendations

### Overall Readiness Status: **READY**

### Finding Summary

| ID | Severity | Story | Description |
|---|---|---|---|
| IR-1 | NOTE | 1.2 | `DailyBriefingSchema` type location not specified — local definition recommended |
| IR-2 | NOTE | 2.2, 2.6 | `ReleaseResult.SmokeTestPassed` assembly implicit — implementor must handle |
| IR-3 | NOTE | 2.1, 2.3–2.6 | Activity constructor struct not defined — define `ReleaseActivities` in Story 2.1 |
| IR-4 | NOTE | 3.1 | `SecretStore` name requires lookup from existing ESO manifests |

**Zero BLOCKER or CRITICAL findings.**
All 4 findings are NOTEs — implementation guidance, not planning gaps.

### Recommended Actions Before Sprint Planning

None blocking. The following are suggested implementation notes to add to sprint context:

1. **Story 1.2**: Define `DailyBriefingSchema` as a local struct in `internal/workflows/daily_briefing.go`
2. **Story 2.1**: Define `ReleaseActivities` struct with all credential fields; make 2.3–2.6 activities methods on this struct
3. **Story 2.2/2.6**: Ensure workflow sets `ReleaseResult{SmokeTestPassed: true}` on `RunSmokeTest` nil return
4. **Story 3.1**: Look up `ClusterSecretStore` name from `fourdogs-central/k8s/` before writing manifests

### Dependency Validation Confirmed

```
1.1 → 1.2 → 1.3 → 1.4   [Epic 1: sequential, correct]
Epic 1 → 2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 2.6   [Epic 2: sequential, correct]
Epic 3: Story 3.1 independent (parallel with Epic 2)   [correct]
1.4 blocked on 1.3   [migration atomicity constraint, adversarial Finding 3, confirmed ✅]
2.3–2.6 Vault pre-conditions explicit   [adversarial Finding 2, confirmed ✅]
```

### Final Note

This assessment identified 4 issues (all NOTEs) across 2 categories (story completeness, implementor guidance). No blockers; no planning gaps. All 12 FRs and 10 NFRs are fully covered. The initiative is **READY** for sprint planning.
