---
verdict: PASS_WITH_NOTES
gate: adversarial-review
gate_mode: advisory
initiative: terminus-infra-crossplane
audience: medium
track: tech-change
reviewer: adversarial-review (party mode)
reviewed_artifacts:
  - docs/terminus/infra/crossplane/architecture.md
reviewed_at: '2026-03-30T18:35:00Z'
constitution_gate: informational
---

# Adversarial Review: terminus-infra-crossplane

**Verdict:** PASS_WITH_NOTES — Architecture is sound for phase 1 homelab scope. All core decisions defensible. Findings below are advisory; none block promotion.

---

## Findings

1. **Provider versions not pinned** — `provider-kubernetes` and `provider-helm` are referenced without specific version tags. The architecture flags this as a GAP but provides no placeholder value or resolution mechanism. An implementation agent will produce non-deterministic manifests. **This is the only implementation blocker — must be resolved before story implementation begins.**

2. **`kustomization.yaml` content unspecified** — Listed as "registers all sub-apps with ArgoCD" but no resource entries, namespace field, or apiVersion are defined. An agent cannot implement this file from the document alone.

3. **ArgoCD Application spec absent** — `Application.yaml` is listed as an artifact but its spec (repoURL, targetRevision, path, syncPolicy, automated/prune) is entirely missing from the document.

4. **`values.yaml` content undefined** — Listed as "Helm values override (if any)" with no guidance on defaults, which fields to override, or whether the file should exist at all.

5. **`revisionHistoryLimit` value unstated** — Agent Consistency Rule 2 mandates setting this on providers but never specifies what value. Agents will invent a number.

6. **InjectedIdentity RBAC requirements unanalyzed** — Document assumes `InjectedIdentity` is zero-setup for `provider-kubernetes`, but the Crossplane ServiceAccount may require specific ClusterRoles for in-cluster object management. This is a known operational issue.

7. **No environment matrix** — Vault paths reference `{env}` and the domain architecture implies dev/prod, but the document never defines which environments are in phase 1 scope.

8. **No rollback/recovery strategy** — A failed provider pod, sick ProviderConfig, or botched Helm install has no documented remediation path.

9. **ArgoCD sync wave ordering omitted** — crossplane-core, providers, and providerconfigs are separate Applications. Ordering (core healthy before providers, providers healthy before providerconfigs) is critical but unaddressed.

10. **No acceptance criteria** — No specification for post-install health verification (provider pod running, `HEALTHY=True` on ProviderConfig, CRDs registered). Without this there are no done criteria for implementation agents.

11. **Self-heal and prune policy not specified** — No guidance on `selfHeal`, `prune`, or reconciliation interval for the ArgoCD Applications.

12. **`terminus.io/managed-by: argocd` label is convention-noise** — Resources synced by ArgoCD labelling themselves as ArgoCD-managed is semantically circular. The label convention adds no routing or filtering value in single-cluster deployments.

---

## Summary

| Severity | Count | Notes |
|----------|-------|-------|
| Implementation blocker | 1 | Finding #1 — provider version pinning (pre-existing known gap) |
| Advisory / gap | 11 | Findings #2–12 — should be addressed in story implementation |

**Gate mode:** advisory (tech-change track, constitution gates informational)
**Promotion verdict:** PASS_WITH_NOTES — proceed to medium→large
