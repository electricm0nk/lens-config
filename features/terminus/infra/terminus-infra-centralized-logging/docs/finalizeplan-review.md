---
feature: terminus-infra-centralized-logging
doc_type: finalizeplan-review
track: tech-change
phase: finalizeplan
verdict: pass-with-conditions
reviewed_at: 2026-04-23T00:00:00Z
---

# FinalizePlan Review — terminus-infra-centralized-logging

## 1. Review Scope

Track: `tech-change` — phases are `techplan` and `finalizeplan` only. Preplan/businessplan artifacts do not apply and are not required.

Planning artifacts under review:
- `architecture.md` (revised 2026-04-23 — post-finalizeplan sensing corrections applied)
- `techplan-adversarial-review.md` (revised 2026-04-23)

---

## 2. Planning Set Assessment

### 2.1 Architecture Completeness

| Area | Assessment |
|------|-----------|
| Storage decision | ✅ Complete — Synology NAS S3 for prod, 20Gi local-path for dev (upgrade staging only) |
| Collector topology | ✅ Complete — single Alloy DaemonSet → prod Loki; no alloy-dev |
| VM log shipping | ✅ Complete — best-effort Alloy service install on each VM, accepted SLA |
| Alert rules | ✅ Complete — 5 rules defined inline via `loki.rulesConfig`; query filters tightened |
| Alert delivery path | ✅ Corrected — Alertmanager already deployed by `prometheus` feature; `AlertmanagerConfig` CRD added in Phase 6 |
| Dev environment | ✅ Complete — loki-dev is upgrade staging only; synthetic data injection; no real Alloy shipping |
| Retention | ✅ Complete — prod 30d with `compactor.retention_enabled: true`; dev 7d no enforcement |
| Secret management | ✅ Complete — Vault KV → ExternalSecret → Kubernetes Secret pattern; no secrets in git |
| Implementation sequence | ✅ Complete — 7 phases including Phase 0 prerequisites; Phase 0 step 3 corrected |
| ArgoCD app topology | ✅ Complete — loki-prod, loki-dev, alloy, alloy-vm, grafana-datasource apps defined |
| Helm values file tree | ✅ Complete — prod/dev overlays, VM config defined |

### 2.2 Techplan Review Disposition

All findings from `techplan-adversarial-review.md` are resolved or accepted:
- **2 Critical**: both resolved
- **6 High**: 5 resolved, 1 (H5 Alertmanager routing coordination) resolved with coordination note in Phase 6
- **3 Medium**: 2 accepted, 1 (M6 Grafana sidecar) — open item carried forward
- **4 Low**: 3 resolved, 1 (L1 loki-dev size) — accepted as 20Gi with upgrade note

**Verdict from techplan review**: `pass-with-warnings`

---

## 3. Cross-Feature Governance Sensing

### 3.1 Affected Features

| Feature | Phase | Impact |
|---------|-------|--------|
| `prometheus` | `complete` | Alertmanager deployed by this feature. Our Phase 6 adds `AlertmanagerConfig` CRD — additive, no conflict risk. |
| `prometheus-wiring` | `finalizeplan-complete` | Owns Alertmanager routing baseline. **Coordination required**: when `prometheus-wiring` enters dev, its Alertmanager config must not conflict with our `AlertmanagerConfig` CRD. Architecture Phase 6 documents migration path. |
| `k3s-startup-sequence` | `dev-complete` | Cluster startup order. No impact — Loki/Alloy are new additions, not involved in existing startup sequence. |
| `proxmox-vm-boot-order` | `finalizeplan-complete` | VM boot order. No impact — VM Alloy agents are post-boot, no boot-order dependency. |
| `secrets` | `preplan` | Vault KV and ESO are prerequisites. Vault and ESO are already operational (used by `prometheus` and others). No blocking dependency. |

### 3.2 Governance Coordination Points

**`prometheus-wiring` coordination (required before dev merge)**

The `prometheus-wiring` feature defines Alertmanager routing using Vault-backed receiver secrets. This feature adds a `AlertmanagerConfig` CRD for Loki ruler alerts. These are parallel delivery mechanisms and can coexist if:
1. Our `AlertmanagerConfig` CRD uses a namespace-scoped selector matching the `monitoring` namespace
2. The global Alertmanager config (`prometheus-wiring`) sets `alertmanager.alertmanagerConfigSelector` to pick up our CRD

**Recommended action**: Add a coordination note to `prometheus-wiring` stories when `prometheus-wiring` enters sprint planning, to ensure the `alertmanagerConfigSelector` is set. No blocking dependency today — `prometheus-wiring` is not yet in dev.

---

## 4. Constitution Gate Check

### 4.1 Applicable Gates (tech-change track)

| Gate | Status |
|------|--------|
| `finalizeplan` | ✅ Pass — architecture reviewed and adversarial review complete; planning set coherent |
| `finalizeplan-to-dev` | ⚠️ Conditions (see §4.2) |

### 4.2 Finalizeplan-to-Dev Conditions

The following conditions must be met before the planning PR merges into the feature branch and this feature advances to dev:

1. **Planning PR open and reviewed**: `terminus-infra-centralized-logging-plan` → `terminus-infra-centralized-logging` (PR #117 at https://github.com/electricm0nk/terminus.infra/pull/117). Requires at least one reviewer pass.

2. **Epics, stories, implementation-readiness, and sprint-status**: must be produced before dev begins (Step 3 of FinalizePlan skill). These artifacts are not yet present.

3. **Phase 0 prerequisites documented in stories**: Synology S3 bucket setup and Vault credential storage must be epics/story steps, not just architecture text. Implementers must not skip Phase 0.

4. **`prometheus-wiring` coordination**: before merging Loki alert rules (Phase 6), ensure `prometheus-wiring` story backlog includes `alertmanagerConfigSelector` configuration. Add as dependency note when `prometheus-wiring` enters sprint planning.

---

## 5. Open Items Carried from TechPlan Review

| ID | Description | Resolution Path |
|----|-------------|----------------|
| H5 | Alertmanager routing coordination with `prometheus-wiring` | Architecture Phase 6 documents `AlertmanagerConfig` CRD approach and migration path. Coordination note required in `prometheus-wiring` stories. |
| H6 | containerd 2.0.4 compatibility with Alloy DaemonSet | Verify during Phase 2 story — Alloy team tracks containerd 2.x compatibility in changelog. If incompatible, pin `containerd` version or use Alloy daemonset restart probe workaround. |
| M6 | Grafana dashboard sidecar reload verification | Verify during Phase 5 story — confirm `sidecar.dashboards.enabled: true` in grafana values and sidecar container running. |

---

## 6. Verdict

**`pass-with-conditions`**

The planning set is coherent, complete, and ready for implementation planning. Architecture corrections applied at finalizeplan sensing round improved governance accuracy. No blocking issues found.

**Conditions before dev merge**:
1. Epics, stories, implementation-readiness, sprint-status artifacts produced (FinalizePlan Step 3)
2. Planning PR #117 reviewed and approved
3. `prometheus-wiring` coordination note added to story backlog (non-blocking today, required before Phase 6 implementation)

The `terminus-infra-centralized-logging` feature is cleared to advance through FinalizePlan Step 2 (planning PR checkpoint) and Step 3 (implementation planning bundle).
