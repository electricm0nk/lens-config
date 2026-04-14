---
initiative: terminus-infra-secrets
phase: devproposal
generatedAt: 2026-03-24T01:30:00Z
stressGateCompletedAt: 2026-03-24T02:10:00Z
stressGateVerdict: PASS_WITH_NOTES
stressGateCompletedAt: 2026-03-24T02:10:00Z
stressGateVerdict: PASS_WITH_NOTES
mode: batch
verdict: READY
---

# Implementation Readiness Checklist — terminus-infra-secrets

**Phase:** DevProposal
**Date:** 2026-03-24
**Reviewer:** John (PM) + Winston (Architect) + Bob (SM)

---

## 1. Planning Artifacts

| Artifact | Path | Status | Notes |
|---|---|---|---|
| PRD | `docs/terminus/infra/secrets/prd.md` | ✅ Complete | 39 FRs, 23 NFRs, 6 journeys |
| Architecture | `docs/terminus/infra/secrets/architecture.md` | ✅ Complete | 8 decisions (D1–D8), full project structure, validation passed |
| Adversarial Review | `docs/terminus/infra/secrets/adversarial-review-report.md` | ✅ PASS_WITH_NOTES | All notes incorporated into stories |
| Epics & Stories | `docs/terminus/infra/secrets/epics.md` | ✅ Complete | 5 epics, 29 stories (incl. 4.7 added post-stress-gate), 38/39 FRs covered |

---

## 2. Requirements Coverage

| Area | FRs | Coverage |
|---|---|---|
| Secret Storage & Delivery | FR1–FR5 | ✅ Epic 2 |
| Identity & Access Control | FR6–FR11 | ✅ Epic 3 |
| Provisioning & Automation | FR12–FR16 | ✅ Epics 1, 3, 4 |
| Audit & Compliance | FR17–FR21 | ✅ Epics 3, 5 |
| Secret Lifecycle Management | FR22–FR26 | ✅ Epics 2, 4 |
| Disaster Recovery & Resilience | FR27–FR32 | ✅ Epic 5 |
| Observability & Health | FR33–FR36 | ✅ Epics 1, 4 |
| Dependency & Patch Management | FR37–FR39 | ✅ FR37-38 in Epic 5; FR39 deferred |

**Total FR coverage: 38/39** — FR39 (CVE automation) intentionally deferred to `terminus-infra-dependency-health` initiative per architecture decision.

> **Stress Gate Note (Epic 2):** `carlpett/sops` provider pin added to Story 1.1 ACs — `carlpett/sops ~> 3.x` required as it is consumed by vault-config data sources.

> **Stress Gate Note:** `carlpett/sops` provider pin has been added to Story 1.1 ACs after Epic 2 stress gate review.

---

## 3. Adversarial Review Story Incorporation

All 9 required stories from the adversarial review are present in epics.md:

| Advisory | Priority | Story | Status |
|---|---|---|---|
| S1 — Vault provider alias (arch complexity) | HIGH | Story 1.6 | ✅ Incorporated |
| S2 — TLS CA cert distribution to clients | HIGH | Story 1.3 + 1.8 | ✅ Incorporated |
| S3 — SOPS state encryption pre-commit guard | HIGH | Story 1.4 | ✅ Incorporated |
| S4 — Isolation verification suite (D8) as first-class | HIGH | Story 3.7 | ✅ Incorporated |
| S5 — Break-glass runbook with exercised run | MEDIUM | Story 5.3 | ✅ Incorporated |
| S6 — Blast-and-repave with exercised run + RTO | MEDIUM | Story 5.1 | ✅ Incorporated |
| S7 — Workload onboarding end-to-end verification | MEDIUM | Story 3.8 | ✅ Incorporated |
| S8 — AppRole token TTL strategy | MEDIUM | Story 3.2 | ✅ Incorporated |
| S9 — Automated rotation stub for Growth | LOW | Story 4.5 | ✅ Incorporated |

---

## 4. Story Quality Checks

### 4.1 — Story sizing
- ✅ Each story is completable by a single operator session or a single dev agent session
- ✅ No story creates more than the tables/resources it needs
- ✅ No story is "set up everything" — each delivers a distinct operator capability

### 4.2 — Dependency flow
- ✅ Epic 1 → Epic 2 → Epic 3: natural progression (infrastructure before secrets before identities)
- ✅ Epic 4 depends on Epic 1 (Ansible needs a running Vault) but is independently executable after
- ✅ Epic 5 depends on Epic 1-3 for blast-and-repave and Epic 4 (snapshots) for restore — dependencies documented in epic goals
- ✅ No story within an epic depends on a future story within the same epic
- ✅ Story 1.4 (SOPS guard) appears before any state is committed (correct ordering)
- ✅ Story 1.6 (provider alias) appears before 1.7 (KV mount) — correct ordering

### 4.3 — Acceptance criteria
- ✅ All stories use Given/When/Then BDD format
- ✅ No stub or placeholder acceptance criteria
- ✅ All ACs are independently testable via CLI commands or Ansible playbooks
- ✅ Security-sensitive ACs (isolation, NFR4 root token revocation) are verifiable

### 4.4 — Architecture compliance
- ✅ No starter template (greenfield infrastructure — correct, no scaffold)
- ✅ OpenTofu state management: SOPS encryption guard is Story 1.4 (early epic placement)
- ✅ Provider boundary separation respected across stories (hashicorp/proxmox only in infra/, hashicorp/vault only in vault-core/ and vault-config/)

### 4.5 — Constitutional compliance
- ✅ Org Article 3: Architecture documentation present (complete architecture.md)
- ✅ Org Article 7/8: BDD acceptance criteria included in every story; TDD discipline applicable to Ansible/OpenTofu test execution
- ✅ Org Article 9: Security-first throughout — secrets never in plaintext, state encryption enforced
- ✅ Terminus Article 1/2: Feature stays within `terminus.infra` repo — no separate repository proposed
- ✅ Infra Article 1/2: All infra features in infra repository — correct

---

## 5. Implementation Prerequisites

| Prerequisite | Status | Notes |
|---|---|---|
| Proxmox environment accessible | Assumed ✅ | Operator responsibility |
| age private key available | Assumed ✅ | NFR6 — offline custody per runbook |
| `terminus.infra` repo exists | Required | Target repo per initiative config |
| `SOPS_AGE_KEY_FILE` env var set | Operator step | Day-0 bootstrap runbook covers this |
| OpenTofu 1.11.0 installed | Required | Toolchain version pinned |
| Ansible-core 2.19 installed | Required | Day-2 operations |
| Snapshot target host configured | Required | NFR10 — separate Proxmox host |

---

## 6. Dependencies and Sequencing

### Required sequence within implementation:
1. **Epic 1** must be fully complete before Epic 2 or 3 can begin (Vault must exist)
2. **Story 1.4** (SOPS guard) should be implemented as the first commit after Story 1.1 (before any state is generated)
3. **Story 3.7** (verify-isolation.yml) **must be interleaved concurrently with Stories 3.3–3.5** — stub the failing checks in 3.3, add k3s check in 3.4, add OpenClaw check in 3.5; do NOT defer 3.7 to end of Epic 3 sprint
4. **Epic 4** (Day-2 ops) can begin in parallel with Epic 3 after Epic 1 is complete
5. **Epic 5** (DR) requires Epic 1 complete, and Story 5.2 requires Epic 4 Story 4.3 (snapshots) to exist

### Known risks:
| Risk | Mitigation |
|---|---|
| Vault provider alias complexity (Story 1.6) | Architecture footnote + story AC includes implementation comment requirement |
| SOPS decryption during tofu apply | `SOPS_AGE_KEY_FILE` env var pattern documented in Day-0 runbook |
| Root token capture from stdout | Story 1.6 AC requires operator capture; Day-0 runbook documents obligation |
| AppRole token expiry during provisioning | Story 3.2 TTL policy documents renewal strategy |

---

## 7. Readiness Verdict

| Check | Status |
|---|---|
| All required planning artifacts exist and are substantive | ✅ |
| 38/39 FRs covered (FR39 intentionally deferred) | ✅ |
| 23/23 NFRs covered | ✅ |
| All adversarial review advisories incorporated | ✅ |
| Epic structure user-value focused (not technical layers) | ✅ |
| Story dependencies flow naturally Epic 1→2→3→4/5 | ✅ |
| All ACs are testable BDD criteria, no stubs | ✅ |
| Constitutional compliance verified (informational gates) | ✅ |

**Verdict: READY FOR SPRINT PLANNING**

This initiative is ready to proceed to `/sprintplan`. The DevProposal artifacts are sufficient for implementation work to begin on Epic 1 in the first sprint.

---

## 8. Epic Stress Gate Summary

Per-epic adversarial review (party mode) executed after generation:

| Epic | Verdict | Notes |
|---|---|---|
| Epic 1: Provisioned Vault Instance | ✅ PASS | No gaps found |
| Epic 2: Secret Authoring Pipeline | ⚠️ PASS_WITH_NOTES | `carlpett/sops` provider pin added to Story 1.1 ACs |
| Epic 3: Workload Identity Management | ⚠️ PASS_WITH_NOTES | Story 3.7 sprint interleave guidance added inline |
| Epic 4: Day-2 Operations | ⚠️ PASS_WITH_NOTES | Story 4.7 (TLS cert renewal procedure) added |
| Epic 5: Disaster Recovery & Resilience | ✅ PASS | No gaps found |

**Post-stress gate story count: 29** (Story 4.7 added). All stress gate notes resolved in `epics.md`. Verdict unchanged: **READY FOR SPRINT PLANNING**.
