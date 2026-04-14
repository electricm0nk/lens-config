---
phase: devproposal
initiative: terminus-infra-semaphoreui
date: 2026-03-28
status: approved
---

# Implementation Readiness Report — terminus-infra-semaphoreui

**Date:** 2026-03-28
**Status:** READY

---

## Summary

All planning artifacts for `terminus-infra-semaphoreui` have been reviewed for implementation readiness. The initiative is cleared to proceed to DevProposal → SprintPlan → Dev.

**Result:** ✅ READY — No blocking issues found.

---

## Artifacts Reviewed

| Artifact | Status | Location |
|---|---|---|
| PRD | ✅ Complete | `docs/terminus/infra/semaphoreui/prd.md` |
| Architecture | ✅ Complete | `docs/terminus/infra/semaphoreui/architecture.md` |
| Tech Decisions | ✅ Complete | `docs/terminus/infra/semaphoreui/tech-decisions.md` |
| Adversarial Review | ✅ PASS_WITH_NOTES | `docs/terminus/infra/semaphoreui/adversarial-review-report.md` |
| Epics & Stories | ✅ Complete | `docs/terminus/infra/semaphoreui/epics.md` |

---

## Readiness Checklist

### 1. Stories Derivable from Architecture?

- [x] Epic 1 (Data Foundation) stories map directly to FR-03 and FR-04
- [x] Epic 2 (Application Delivery) stories map directly to FR-01, FR-02, FR-05
- [x] Epic 3 (ArgoCD Access) stories map directly to FR-06
- [x] All 6 stories have testable acceptance criteria
- [x] No stories have ambiguous scope

**Result:** ✅ PASS

---

### 2. All Dependencies Identified?

- [x] Patroni Postgres HA cluster operational — `10.0.0.56:5432` ✅ (from k3s initiative)
- [x] Vault operational with KV v2 at `secret/terminus/infra/` ✅ (from secrets initiative)
- [x] ESO ClusterSecretStore `vault-backend` functional ✅ (from k3s initiative)
- [x] cert-manager ClusterIssuer `vault-pki` operational ✅ (from k3s initiative)
- [x] `local-path` StorageClass available ✅ (from k3s initiative)
- [x] Traefik LoadBalancer VIP `10.0.0.126` active ✅ (from k3s initiative)
- [x] ArgoCD App-of-Apps running ✅ (from k3s initiative)
- [ ] DNS `semaphore.trantor.internal → 10.0.0.126` — **operator prerequisite, pending** (Story 2.2)
- [ ] DNS `argocd.trantor.internal → 10.0.0.126` — **operator prerequisite, pending** (Story 3.1)
- [ ] Vault KV `secret/terminus/infra/semaphoreui` seeded — **operator prerequisite, pending** (Story 1.2)

**Result:** ✅ PASS — All platform dependencies are in place. Pending items are expected operator-manual prerequisites documented in stories.

---

### 3. No Plaintext Secrets in Any Artifact?

- [x] PRD: no credentials
- [x] Architecture: all credential references are Vault paths or k8s Secret names only
- [x] Tech Decisions: TDs reference Vault paths, no values
- [x] Epics: story tasks mention vault paths and field names, no actual secrets

**Result:** ✅ PASS

---

### 4. Implementation Path Clear (No Blockers)?

- [x] Vault KV path `secret/terminus/infra/semaphoreui` is defined (TD-008)
- [x] Tofu module `postgres-databases` exists and is the established pattern
- [x] Manifest file layout is fully specified in architecture.md
- [x] ArgoCD insecure mode approach is documented with restart step (TD-018)
- [x] Deployment sequence order (10 steps) is documented — no circular dependencies
- [x] ArgoCD bootstrapping problem for its own ingress is documented and resolved (manual kubectl)

**Result:** ✅ PASS

---

### 5. Architecture Consistent with PRD?

| PRD Requirement | Architecture Support | Status |
|---|---|---|
| Semaphore at `semaphore.trantor.internal` | Ingress + Certificate in architecture.md | ✅ |
| ArgoCD at `argocd.trantor.internal` | Epic 3, TD-019 | ✅ |
| Credentials from Vault via ESO | ExternalSecret spec in architecture.md | ✅ |
| Postgres DB provisioned via Tofu | Standalone Tofu env, TD-012 | ✅ |
| ArgoCD automated sync (prune + selfHeal) | TD-016, Story 2.3 | ✅ |
| 1Gi PVC, `local-path` | TD-007, Story 2.1 | ✅ |

**Note:** Vault path inconsistency between PRD (`semaphore`) and architecture (`semaphoreui`) is resolved by TD-008. All implementation stories use `semaphoreui`.

**Result:** ✅ PASS

---

### 6. Per-Epic Adversarial + Readiness Gate

#### Epic 1: Data Foundation

- [x] Story 1.1 (Tofu DB): acceptance criteria are testable, no circular dependencies, Tofu pattern established
- [x] Story 1.2 (Vault/ESO): acceptance criteria testable with `kubectl get externalsecret`, dependency on Vault seeding documented
- **Adversarial finding:** None blocking. The DB password value must be agreed between Tofu var and Vault KV before apply — order matters (seed Vault first, then tofu apply).
- **Verdict:** ✅ PASS

#### Epic 2: Semaphore Application Delivery

- [x] Story 2.1 (Deployment/Service/PVC): readiness probe health endpoint noted as verify-needed — acceptable implementation detail
- [x] Story 2.2 (Ingress/Cert): DNS prerequisite documented; cert issuance failure path documented
- [x] Story 2.3 (ArgoCD App): App-of-Apps sync behavior documented, smoke test login step included
- **Adversarial finding:** `SEMAPHORE_DB_PASS` injection method — using both `envFrom: secretRef` AND explicit `valueFrom: secretKeyRef` in the same Deployment spec for the same Secret is redundant. Use one method: `envFrom` for all fields from `semaphoreui-secrets`. Implementation story should confirm this.
- **Verdict:** ✅ PASS (with minor implementation note above)

#### Epic 3: ArgoCD Platform Access

- [x] Story 3.1: insecure mode + restart step documented, port `80` (not `443`) for ingress backend is explicit, manual kubectl rationale is explicit
- **Adversarial finding:** The redirect loop risk (using port 443 in ingress behind insecure ArgoCD) is documented with explicit resolution guidance. Acceptable.
- **Verdict:** ✅ PASS

---

### 7. Security Gate

- [x] No credentials in manifests
- [x] All external endpoints use TLS via `vault-pki`
- [x] Semaphore Service is ClusterIP — no direct external exposure
- [x] `SEMAPHORE_ACCESS_KEY_ENCRYPTION` is a randomized 32+ char string from Vault — Semaphore secrets are encrypted at rest
- [x] ArgoCD insecure mode only affects internal cluster traffic — external TLS is maintained via Traefik

**Result:** ✅ PASS

---

### 8. Story Count and Scope Validation

| Epic | Stories | Estimated Complexity |
|---|---|---|
| Epic 1: Data Foundation | 2 | Medium (Tofu + Vault manual seeding) |
| Epic 2: Semaphore Application Delivery | 3 | Medium (manifest authoring + ArgoCD registration) |
| Epic 3: ArgoCD Platform Access | 1 | Low-Medium (ConfigMap patch + manifests + apply) |
| **Total** | **6** | **Medium overall** |

Single sprint delivery is feasible. All stories have clear done criteria.

---

## Outstanding Notes (Non-Blocking)

The following notes from the adversarial review should be reflected in implementation but do not block DevProposal:

1. **Vault path `semaphoreui` vs `semaphore`** — TD-008 is the canonical source. Use `semaphoreui` everywhere.
2. **Patroni direct-IP binding** — `10.0.0.56` is fragile if primary changes. Document as accepted homelab constraint in Story 2.1 notes.
3. **`envFrom` vs `valueFrom`** — Use `envFrom: secretRef: semaphoreui-secrets` exclusively. No duplicate `valueFrom` for `SEMAPHORE_DB_PASS`.
4. **Smoke test** — Story 2.3 includes a login smoke test. A formal `validate-semaphoreui.sh` script (parallel to k3s `validate-contract.sh`) is recommended but not required for first delivery.
5. **ArgoCD non-GitOps manifests** — Clearly comment `platforms/k3s/manifests/argocd/` files as manually applied to prevent future operators from adding them to the App-of-Apps.

---

## Conclusion

✅ **Implementation Readiness: READY**

All 6 stories across 3 epics are sufficiently specified for implementation to begin. Platform prerequisites are met. Operator manual prerequisites are documented. No blocking architectural issues found.
