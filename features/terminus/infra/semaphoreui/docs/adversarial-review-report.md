---
verdict: PASS_WITH_NOTES
phase: devproposal-entry-gate
initiative: terminus-infra-semaphoreui
date: 2026-03-28
reviewers: [analyst, architect, pm, qa, ux-designer]
artifacts_reviewed:
  - docs/terminus/infra/semaphoreui/prd.md
  - docs/terminus/infra/semaphoreui/architecture.md
  - docs/terminus/infra/semaphoreui/tech-decisions.md
---

# Adversarial Review Report — terminus-infra-semaphoreui

**Gate:** small → medium audience promotion (entry gate: adversarial-review, party mode)
**Date:** 2026-03-28
**Verdict:** PASS_WITH_NOTES

---

## Review Panel

| Reviewer | Role | Perspective |
|---|---|---|
| Mary | Business Analyst | Requirements completeness, user value, scope clarity |
| Winston | Architect | Technical feasibility, design soundness, implementation readiness |
| John | PM | Story derivability, delivery risk, success criteria |
| Quinn | QA Engineer | Testability, validation coverage, observable success |
| Sally | UX Designer | Usability, access patterns, operator experience |

---

## Executive Summary

The planning artifacts for `terminus-infra-semaphoreui` are well-constructed and consistent across prd.md, architecture.md, and tech-decisions.md. The initiative has clear business value (first workload on k3s platform, establishes reference pattern), a well-defined technical scope with 20 documented decisions, and a coherent deployment sequence. All reviewers consider the artifacts sufficient to proceed to DevProposal.

Notable strengths:
- Architecture is operationally grounded with a concrete 10-step deployment sequence
- All secrets handled through Vault → ESO — no plaintext credentials pattern
- ArgoCD insecure mode via ConfigMap (not Deployment mutation) is the correct approach
- Tofu environment separation is well-reasoned

Notes (non-blocking) are recorded below.

---

## Reviewer Findings

### Mary (Business Analyst) — Requirements Completeness

**Verdict:** PASS

**Strengths:**
- PRD clearly articulates the "first workload gap" and why Semaphore UI is the right choice
- Success criteria are measurable with specific URLs, TLS status, and behavior expectations
- Scope boundary (out-of-scope: SSO, HA, CI, monitoring) is explicit — prevents scope creep
- User journeys are realistic and trace back to concrete functional requirements

**Notes:**
- The PRD references `secret/terminus/infra/semaphore/` while architecture.md uses `secret/terminus/infra/semaphoreui/` (TD-008 explicitly supersedes the PRD reference). This discrepancy is noted but TD-008 makes the canonical path clear — no blocker.
- Admin email is required by Semaphore but not mentioned in user journeys (though it's in the ExternalSecret fieldset). Acceptable — it's a bootstrap detail, not a user-facing requirement.

---

### Winston (Architect) — Technical Feasibility

**Verdict:** PASS

**Strengths:**
- Per-resource manifest split is correct for ArgoCD/GitOps — resource-level sync status visibility
- `imagePullPolicy: Always` is the right choice for `latest` tag — correctly rationalized
- `replicas: 1` constraint driven by `ReadWriteOnce` PVC is architecturally sound
- Explicit Certificate objects (not annotation-driven) are easier to debug and monitor
- TD-018 (ArgoCD insecure mode via ConfigMap) is the canonical approach and avoids Deployment mutation drift
- Deployment sequence (Step 8 note) correctly identifies the ArgoCD bootstrapping problem for its own ingress

**Notes:**
- The Patroni primary is referenced by direct IP (`10.0.0.56`). If Patroni promotes a new primary to a different VIP, this will break the connection. For homelab this is acceptable, but the implementation story should document the constraint and recommend a DNS name or pgBouncer as a future improvement.
- No readiness probe details are specified in the architecture (only "HTTP GET on port 3000 at `/api/ping` or root"). The story author should verify the actual Semaphore health endpoint. Minor — implementation detail.
- The Tofu state key `tofu/postgres/semaphore` is specified but Consul connection details are not — the implementation story must reference the existing Consul backend config. Not a blocker.

---

### John (PM) — Story Derivability

**Verdict:** PASS

**Strengths:**
- PRD high-level story breakdown (6 stories) aligns cleanly with architecture sections
- All functional requirements (FR-01 through FR-06) map to distinct, implementable units
- Dependencies and preconditions table is complete with status indicators
- ArgoCD ingress is explicitly in scope with FR-06 — no ambiguity about what's included

**Notes:**
- Story 6 (ArgoCD Ingress) has a manual deployment step (kubectl, not GitOps) that will need careful story acceptance criteria. The story author must distinguish the GitOps-managed resources from the manually applied ArgoCD manifests.
- The 10-step deployment sequence in architecture.md implies an operator runbook. Make sure the implementation story for bootstrap covers user-facing documentation, not just manifest creation.

---

### Quinn (QA Engineer) — Testability

**Verdict:** PASS

**Strengths:**
- Success criteria in the PRD are directly testable (URL loads, TLS valid, creds from Vault)
- Validation approach is straightforward: browser test + curl for TLS inspection
- Contract validation pattern from k3s (`validate-contract.sh`) can be extended to include Semaphore availability checks post-deployment

**Notes:**
- No automated test or health check validation is included in the current architecture. For a homelab workload this is acceptable, but the implementation stories should include a basic smoke test script (similar to k3s `validate-contract.sh`) covering:
  - `curl -k https://semaphore.trantor.internal` returns 200
  - `curl -k https://argocd.trantor.internal` returns 200
  - ExternalSecret is synced (`kubectl get secret semaphoreui-secrets`)
  - PVC is bound (`kubectl get pvc semaphoreui-data`)
- Not a blocker, but recommended as an epic/story requirement.

---

### Sally (UX Designer) — Operator Experience

**Verdict:** PASS

**Strengths:**
- Journey 1 (run playbook via UI) is clear and realistic
- Journey 2 (bootstrap config) correctly identifies the admin credential seeding as a one-time manual prerequisite — this is the right UX decision
- The separation of "operator prerequisites" from automated deployment is clean

**Notes:**
- The operator has to seed Vault manually before any automated step can run. A checklist or runbook for this step would improve the bootstrap experience. Architecture mentions it but doesn't detail the exact `vault kv put` command structure — the implementation story should include this.
- ArgoCD UI access at `argocd.trantor.internal` directly improves day-to-day operator workflow — this is high-value and the PRD rationale is correct.

---

## Cross-Cutting Notes

1. **Vault path inconsistency (PRD vs architecture):** TD-008 resolves this in favor of `semaphoreui`. Implementation stories must use `secret/terminus/infra/semaphoreui` — no story should reference `semaphore` (singular).

2. **Patroni direct-IP binding:** Using `10.0.0.56` directly is a known fragility. Document in story acceptance criteria that this is a homelab constraint and create a tracking note for future DNS-name migration.

3. **ArgoCD bootstrapping (Step 8 sequence):** The kubectl-apply path for ArgoCD ingress/certificate must be documented clearly in the story as non-GitOps. This prevents operator confusion.

4. **Smoke test script:** Consider creating a `validate-semaphoreui.sh` parallel to `validate-contract.sh` to formally close out deployment. Can be a simple checklist bash script.

---

## Verdict Summary

| Reviewer | Domain | Verdict |
|---|---|---|
| Mary | Business Analysis | PASS |
| Winston | Architecture | PASS |
| John | Product Management | PASS |
| Quinn | QA | PASS |
| Sally | UX | PASS |

**Overall:** ✅ **PASS_WITH_NOTES** — Proceed to DevProposal. Notes above are informational and should be reflected in implementation stories but do not block planning advancement.
