---
feature: k3s-startup-sequence
doc_type: finalizeplan-review
status: approved
goal: "Add deterministic ArgoCD sync-wave startup sequencing so k3s pod load order is predictable and dependency ordering is enforced on cluster restart"
key_decisions:
  - Annotation-only changes — no Helm values or chart modifications required
  - 8-wave design (0–8) with explicit infra-namespace, temporal, runner, and ingress tiers
  - Dev apps in scope — same cluster, same wave rules apply
  - Influxdb and Ollama treated as active workloads in wave 6
  - Wave compliance verified via kubectl custom-column command in story ACs
open_questions: []
depends_on: []
blocks: []
updated_at: "2026-04-19"
---

# FinalizePlan Review — k3s-startup-sequence

**Verdict:** pass
**Phase:** finalizeplan
**Source:** phase-complete

---

## Combined Planning Set Assessment

**Artifacts reviewed:**
- `architecture.md` — complete, 8-wave design, 30-file change table, 4 ADRs resolved
- `techplan-adversarial-review.md` — pass-with-warnings; both high findings carried into story ACs

**Track:** `tech-change` — no BusinessPlan/PrePlan predecessors required. Architecture is the sole planning artifact and is sufficient.

---

## Cross-Artifact Coherence

✅ Architecture goal, scope, and implementation approach are internally consistent
✅ All critical findings from TechPlan adversarial review are resolved
✅ High findings (rollback procedure, post-merge smoke test) are tracked in story ACs
✅ 30-file change set is annotation-only — no chart or value changes required
✅ Wave assignments derived from documented dependency graph, not arbitrary ordering

---

## Findings Resolved in This Phase

### M-01 — Rollback procedure
**Resolution:** Rollback is a `git revert` of the wave-annotation commit. Story ACs will include the revert command and note that cluster state is unaffected — ArgoCD sync-waves are ArgoCD-side metadata only.

### M-02 — Post-merge smoke test gap
**Resolution:** Each story AC includes a kubectl wave-compliance verification command that validates the annotation values are correct after ArgoCD syncs. This serves as the post-merge smoke gate.

---

## Blind-Spot Challenge Resolutions (Yolo Defaults)

| # | Question | Resolution |
|---|----------|-----------|
| 1 | Should SemaphoreUI be wave 5 or wait for a separate decision? | Wave 5 — deliberate sequencing after actions-runner, before workloads; inline-secret pattern is expected for this app |
| 2 | Are dev variants always deployed? Should they be skipped in the initial story? | Dev variants in scope — same cluster, treated identically to prod variants in this k3s dev cluster |
| 3 | What is the observable acceptance test for the full wave sequence? | `kubectl` wave-compliance command in ACs; optional: ArgoCD UI observation after force-sync |
| 4 | Does Temporal have any downstream dependency that would require a wave-4 or wave-5 placement? | No — workloads are not block-waiting on Temporal. Wave 2/3 for Temporal is correct |
| 5 | Should InfluxDB and Ollama be treated as active production workloads or optional/dev apps? | Active — wave 6 placement is intentional; no opt-out |

---

## Acceptance Criterion: Wave Compliance Verification

**Observable evidence:** After ArgoCD syncs with updated annotations, all Application objects carry the expected `argocd.argoproj.io/sync-wave` values.

**Operator validation command (included verbatim in story ACs):**
```bash
kubectl get applications -n argocd \
  -o custom-columns=\
'NAME:.metadata.name,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave' \
  --sort-by='.metadata.annotations.argocd\.argoproj\.io/sync-wave'
```

Applications with no annotation will appear at the top (wave 0 / primitives tier — expected for metallb, cert-manager, external-secrets, traefik, crossplane).
