---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/terminus/architecture.md
  - _bmad-output/lens-work/initiatives/terminus/infra/crossplane.yaml (via git)
workflowType: 'architecture'
lastStep: 8
status: 'complete'
project_name: 'terminus-infra-crossplane'
user_name: 'CrisWeber'
date: '2026-03-30'
completedAt: '2026-03-30'
initiative_root: 'terminus-infra-crossplane'
track: 'tech-change'
audience: 'small'
---

# Architecture Decision Document
## terminus-infra-crossplane

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context Analysis

### Initiative Overview

**Scope:** Install and configure Crossplane as a GitOps infrastructure operator on the k3s cluster within the Terminus domain.

**Track:** tech-change — no PRD required. Architecture is the primary deliverable.

**Dependencies:**
- `terminus-infra-k3s` — Crossplane runs as a k3s workload
- `terminus-infra-secrets` — Vault KV v2 for future cloud provider credentials

### Architectural Context

Crossplane sits in the `terminus-infra` namespace on the k3s cluster, managed by ArgoCD (App-of-Apps pattern). It provides a GitOps-native operator abstraction for managing infrastructure resources declaratively from Kubernetes manifests in the `terminus.infra` git repo.

**Delivery chain:**
```
terminus.infra git repo → ArgoCD → k3s → Crossplane + Providers
```

### Phase 1 Scope Decision

**Self-hosted only.** No cloud provider integration in this phase. Design is extensible — cloud providers can be added without refactoring by adding new ProviderConfig entries.

### Provider Selection

| Provider | Purpose | Phase |
|----------|---------|-------|
| `provider-kubernetes` | Manage k8s objects declaratively | Phase 1 ✅ |
| `provider-helm` | Deploy Helm charts as CRDs | Phase 1 ✅ |
| `provider-aws` / cloud | Cloud resource provisioning | Future 🔲 |
| `provider-nop` | Testing/dry-run | Optional 🔲 |

### Credential Strategy

| Provider | Credential Source | Rationale |
|----------|-----------------|-----------|
| `provider-kubernetes` | `InjectedIdentity` (in-cluster SA) | No external creds needed for in-cluster ops |
| `provider-helm` | `InjectedIdentity` (in-cluster SA) | No external creds needed for in-cluster Helm |
| Future cloud providers | Vault KV `secret/terminus/{env}/crossplane/{provider}/` | Reserved, not used in phase 1 |

**Decision:** Phase 1 requires **no Vault wiring for provider credentials** — `InjectedIdentity` covers all in-cluster operations. Vault path reserved for extensibility.

### Scale & Complexity

- **Complexity:** Medium — GitOps-native k8s operator with future extensibility
- **Primary domain:** Infrastructure-as-code / platform engineering
- **Cross-cutting concerns:** ArgoCD sync, k3s RBAC, Vault path design

---

## Starter Template

**Selected:** Official Crossplane Helm chart (`crossplane-stable/crossplane`) wrapped in an ArgoCD `Application` manifest.

**Rationale:** Aligns with existing GitOps delivery pattern (ArgoCD App-of-Apps). No custom operator distribution needed for self-hosted scope.

**Delivery pattern:**
```
terminus.infra git repo
└── apps/
    └── crossplane/
        ├── Application.yaml       (ArgoCD Application pointing to Helm chart)
        ├── providers/
        │   ├── provider-kubernetes.yaml
        │   └── provider-helm.yaml
        └── providerconfigs/
            ├── kubernetes-providerconfig.yaml
            └── helm-providerconfig.yaml
```

**Not selected:** Upbound Universal Crossplane (UXP) — more opinionated, unnecessary for self-hosted scope.

---

## Core Architectural Decisions

### Decision 1 — Namespace Strategy
**Decision:** `crossplane-system` (default upstream namespace)  
**Rationale:** Conventional, matches all upstream documentation, keeps Crossplane self-contained. Reduces cognitive overhead when debugging.

### Decision 2 — Package Registry
**Decision:** Upbound Marketplace (`xpkg.upbound.io`) directly  
**Rationale:** No local registry mirror needed for self-hosted home lab with 2 providers. Can be revisited if air-gap requirements emerge.

### Decision 3 — RBAC Strategy
**Decision:** Default install (Crossplane manages its own RBAC)  
**Rationale:** Internal home lab cluster — broad permissions acceptable for phase 1. Minimal RBAC hardening deferred to a future initiative.

### Decision 4 — Composition Strategy
**Decision:** Raw managed resources (`HelmRelease`, `Object` CRDs directly) — no Compositions/XRDs  
**Rationale:** Compositions are valuable for multi-team self-service platforms, not a solo home lab. Over-engineering at this stage. Can be introduced later if platform-as-a-service patterns emerge.

### Deferred Decisions
| Decision | Deferred Until |
|----------|---------------|
| Cloud provider integration | Future initiative — not phase 1 |
| Local registry mirror | Only if air-gap requirements emerge |
| Compositions / XRDs | Only if multi-team platform patterns emerge |
| Minimal RBAC hardening | Future security hardening initiative |

### Sync Wave Ordering (resolved — Story 3.3)

Crossplane requires strict installation ordering. ArgoCD sync waves enforce this:

| Wave | Resource | Reason |
|------|----------|--------|
| 0 | `crossplane-core/Application.yaml` | Installs Crossplane Helm chart. CRDs must exist before Provider CRDs can be applied. |
| 1 | `providers/provider-kubernetes.yaml` | Crossplane must be HEALTHY. Creates Provider SA + installs provider controller. |
| 1 | `providers/provider-helm.yaml` | Same as above — co-wave with provider-kubernetes. |
| 2 | `providerconfigs/kubernetes-default.yaml` | Provider must be HEALTHY and its CRDs registered. |
| 2 | `providerconfigs/helm-default.yaml` | Same as above. |
| 2 | `crossplane-core/rbac-provider-kubernetes.yaml` | Co-wave — SA should exist by wave 2. |

**Applied via:** `argocd.argoproj.io/sync-wave: "{n}"` annotation on each manifest.

**Parent Application settings** (`platforms/k3s/argocd/apps/crossplane.yaml`):
- `syncPolicy.automated.prune: true` ✅
- `syncPolicy.automated.selfHeal: true` ✅
- `syncOptions: [CreateNamespace=true, ServerSideApply=true]` ✅

**crossplane-core Application settings** (`apps/crossplane/crossplane-core/Application.yaml`):
- `syncPolicy.automated.prune: true` ✅
- `syncPolicy.automated.selfHeal: true` ✅
- `syncOptions: [CreateNamespace=true, ServerSideApply=true]` ✅

---

## Implementation Patterns & Consistency Rules

### Naming Conventions

| Category | Convention | Example |
|----------|------------|---------|
| Provider package | `upbound/provider-{name}` | `upbound/provider-kubernetes` |
| ProviderConfig name | `default` (single cluster) | `spec.providerConfigRef.name: default` |
| Crossplane namespace | `crossplane-system` | all core CRDs |
| ArgoCD Application name | `crossplane-{component}` | `crossplane-core`, `crossplane-providers` |
| Managed resource name | `{service}-{purpose}` | `temporal-helmrelease` |
| Manifest file name | `{kind}-{name}.yaml` (lowercase kebab-case) | `provider-kubernetes.yaml` |

### Structural Rules

- **One resource per file** — no multi-document YAML files
- **All Crossplane manifests live in `terminus.infra` repo** under `apps/crossplane/`
- **ArgoCD App-of-Apps** is the only delivery mechanism — no `kubectl apply` directly

### Agent Consistency Rules

1. **Never use `latest` tag** for provider packages — always pin a specific version
2. **Always set `revisionHistoryLimit`** on providers to avoid unbounded revision accumulation
3. **All managed resources reference `providerConfigRef.name: default`** unless explicitly multi-cluster
4. **Required labels on all managed resources:**
   ```yaml
   labels:
     app.kubernetes.io/part-of: crossplane
     terminus.io/managed-by: argocd
   ```
5. **No inline secrets in manifests** — credentials via `InjectedIdentity` or future ESO ExternalSecrets only

---

## Project Structure

### Repository: `terminus.infra`

```
terminus.infra/
└── apps/
    └── crossplane/
        ├── kustomization.yaml                    # registers all sub-apps with ArgoCD
        │
        ├── crossplane-core/
        │   ├── Application.yaml                  # ArgoCD Application — installs Helm chart
        │   └── values.yaml                       # Helm values override (if any)
        │
        ├── providers/
        │   ├── provider-kubernetes.yaml          # Provider CRD — upbound/provider-kubernetes
        │   └── provider-helm.yaml                # Provider CRD — upbound/provider-helm
        │
        └── providerconfigs/
            ├── kubernetes-default.yaml           # ProviderConfig name: default, InjectedIdentity
            └── helm-default.yaml                 # ProviderConfig name: default, InjectedIdentity
```

### Component Boundaries

| Boundary | Owner | Notes |
|----------|-------|-------|
| Crossplane core install | ArgoCD Application | Helm chart from `crossplane-stable` |
| Provider lifecycle | Crossplane operator | `Provider` CRDs manage install/upgrade |
| ProviderConfig | ArgoCD Application | Declarative, committed to git |
| Managed resources | Individual feature initiatives | e.g., `terminus-platform-temporal` creates `HelmRelease` |

### Integration Boundaries

**Crossplane → k3s:** Runs in `crossplane-system`, uses in-cluster ServiceAccount via `InjectedIdentity`. ArgoCD sync watches `terminus.infra/apps/crossplane/**`.

**Consumer initiatives → Crossplane:** Future initiatives create managed resources (`HelmRelease`, `Object`) referencing `providerConfigRef.name: default`.

**Crossplane → Vault (future):** ESO `ExternalSecret` → k8s Secret → `ProviderConfig.credentialsSecretRef`. Vault path: `secret/terminus/{env}/crossplane/{provider}/credentials`.

---

## Architecture Validation

### Coherence Checks

| Check | Result | Notes |
|-------|--------|-------|
| Provider selection matches scope | ✅ | `provider-kubernetes` + `provider-helm` covers all phase 1 managed resources |
| Credential strategy consistent | ✅ | `InjectedIdentity` used uniformly — no mixed strategies |
| Namespace aligned with conventions | ✅ | `crossplane-system` matches Terminus infrastructure naming |
| Repo structure matches delivery model | ✅ | ArgoCD App-of-Apps pattern; `terminus.infra/apps/crossplane/` layout complete |
| No Compositions in scope | ✅ | Explicitly deferred — raw managed resources only |
| Vault path reserved but not required | ✅ | Path defined for future ESO integration, not blocking phase 1 |

### Requirements Coverage

| Requirement | Coverage | Notes |
|-------------|----------|-------|
| Crossplane installed on k3s | ✅ | Helm chart via ArgoCD Application |
| GitOps delivery | ✅ | All resources committed, ArgoCD sync |
| Provider-kubernetes enabled | ✅ | Provider CRD + ProviderConfig |
| Provider-helm enabled | ✅ | Provider CRD + ProviderConfig |
| No Vault dependency (phase 1) | ✅ | InjectedIdentity eliminates credential management |
| Consumer initiative contract defined | ✅ | `providerConfigRef.name: default` convention established |

### Implementation Readiness

| Gate | Status | Notes |
|------|--------|-------|
| Architecture decisions final | ✅ | 4 decisions locked, no open questions |
| File structure defined | ✅ | All 6 files specified with exact paths |
| Naming conventions documented | ✅ | `terminus-io/{provider}` pattern and label rules |
| Consistency rules documented | ✅ | 5 agent rules ready for implementation guidance |
| Crossplane Helm chart version | ⚠️ | **GAP:** Pin before implementation — `helm search repo crossplane-stable/crossplane` |
| Provider versions | ⚠️ | **GAP:** Pin before implementation — check Upbound Marketplace for `provider-kubernetes` and `provider-helm` |

### Validation Summary

Architecture is **ready for implementation** with one pre-implementation task: verify and pin exact chart/provider versions before committing manifests.

---

## Architecture Complete

This document represents the complete technical architecture for `terminus-infra-crossplane`. All decisions have been made collaboratively, validated for coherence, and are ready to serve as the single source of truth for implementation.

**Pre-implementation required:** Run `helm search repo crossplane-stable/crossplane` and check [marketplace.upbound.io](https://marketplace.upbound.io) for `upbound/provider-kubernetes` and `upbound/provider-helm` to pin exact versions in your `Provider` CRDs and ArgoCD Application.

---

## Pinned Versions
<!-- Updated: 2026-03-29 — verified via Upbound Marketplace and charts.crossplane.io -->

| Component | Version | Full Reference | Registry | Verified |
|---|---|---|---|---|
| Crossplane core | v2.2.0 | `crossplane-stable/crossplane:2.2.0` | charts.crossplane.io/stable | 2026-03-29 |
| provider-kubernetes | v1.2.2 | `xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2` | Upbound Marketplace | 2026-03-29 |
| provider-helm | v1.2.1 | `xpkg.upbound.io/upbound/provider-helm:v1.2.1` | Upbound Marketplace | 2026-03-29 |

**Notes:**
- Both providers require **Crossplane 2.0+** — v2.2.0 satisfies this.
- Both providers are Upbound-signed, CVE-remediated, 12-month support window (until 2027-03-25).
- Use `upbound/` namespace on Marketplace — not `crossplane-contrib/` (older/community versions).
- Update these versions before re-implementing in a new environment. Check Marketplace before pinning.

---

## RBAC Notes

### provider-kubernetes — InjectedIdentity ClusterRoleBinding (REQUIRED)

`InjectedIdentity` for `provider-kubernetes` is **NOT zero-setup**. The Crossplane ServiceAccount
for the provider pod requires an explicit ClusterRoleBinding to manage Kubernetes objects
cross-namespace. Without this, `Object` managed resources will fail to reconcile with permission
errors.

**Required post-install step (manual — SA name is dynamic):**
```bash
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
  sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount="${SA}"
```

This is addressed in Story 2.3 via a `ClusterRoleBinding` manifest using a label selector.
The SA name is dynamic (includes a Crossplane-generated hash) so the binding uses a
predictable label: `app.kubernetes.io/name: upbound-provider-kubernetes`.

**Source:** Upbound Marketplace docs for `provider-kubernetes` v1.2.2.

### provider-helm — InjectedIdentity

`provider-helm` in-cluster does **not** require additional ClusterRoleBinding beyond the default
Crossplane installation. The Helm SA operates within the cluster's existing RBAC without
cross-namespace writes. Confirm during Story 2.3 smoke test.
