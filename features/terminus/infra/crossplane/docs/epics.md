# Epics — terminus-infra-crossplane

**Track:** tech-change  
**Source:** Architecture Decision Document + Adversarial Review (PASS_WITH_NOTES)  
**Sprint Planning by:** Bob (Scrum Master)  
**Date:** 2026-03-30  
**Phase:** SprintPlan (sprintplan, large audience)

---

> **Note:** This initiative follows the tech-change track. Epics and stories are derived directly
> from the architecture document and adversarial review findings. There is no preceding devproposal
> phase. Stories are ordered for safe sequential implementation.

---

## Epic 1: Crossplane Core Installation

**Goal:** Install Crossplane via ArgoCD App-of-Apps into the k3s cluster under `crossplane-system`.

**Scope:**
- Resolve the pre-implementation version-pinning blocker (adversarial finding #1)
- Create ArgoCD `Application.yaml` for the Crossplane Helm chart
- Configure `values.yaml` overrides
- Author `kustomization.yaml` to register all sub-apps with ArgoCD

---

### Story 1.1: Research and Pin Crossplane Provider Versions

**Priority:** BLOCKER — must be completed first  
**Source:** Adversarial finding #1 (implementation blocker)

**Context:**  
The architecture references `upbound/provider-kubernetes` and `upbound/provider-helm` without specific version tags. The Agent Consistency Rule 1 mandates: _Never use `latest` — always pin a specific version_. Without pinned versions, an implementation agent will produce non-deterministic manifests.

**Acceptance Criteria:**
- [ ] Research current stable release of `provider-kubernetes` from Upbound Marketplace
- [ ] Research current stable release of `provider-helm` from Upbound Marketplace
- [ ] Document pinned versions in architecture.md under a new "Pinned Versions" section
- [ ] Confirm versions are accessible from `xpkg.upbound.io` (Upbound Marketplace)

**Implementation Notes:**
- Check: https://marketplace.upbound.io/providers/upbound/provider-kubernetes
- Check: https://marketplace.upbound.io/providers/upbound/provider-helm
- Pin format: `xpkg.upbound.io/upbound/provider-kubernetes:vX.Y.Z`

---

### Story 1.2: Create crossplane-core ArgoCD Application

**Priority:** High — core delivery mechanism  
**Source:** Architecture: `terminus.infra/apps/crossplane/crossplane-core/Application.yaml`  
**Depends-on:** Story 1.1 (versions pinned), working k3s cluster with ArgoCD

**Context:**  
The Application.yaml bootstraps the Crossplane Helm chart install via ArgoCD. The spec must define `repoURL`, `targetRevision`, `path`, `syncPolicy`, `automated/prune`, and `selfHeal`. Adversarial finding #3 flagged these as absent from the architecture. This story fills that gap.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/crossplane-core/Application.yaml`
- [ ] `spec.source.repoURL` points to `https://charts.crossplane.io/stable`
- [ ] `spec.source.chart: crossplane` with pinned `targetRevision`
- [ ] `spec.destination.namespace: crossplane-system`
- [ ] `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
- [ ] ArgoCD sync wave annotation set: `argocd.argoproj.io/sync-wave: "0"` (sync ordering)
- [ ] File passes `kubectl apply --dry-run=client -f Application.yaml`

**Implementation Notes:**
- Sync wave 0 = crossplane-core (must be healthy before providers spin up)
- Crossplane Helm registry: `crossplane-stable` → `https://charts.crossplane.io/stable`
- Required label: `app.kubernetes.io/part-of: crossplane`

---

### Story 1.3: Create crossplane-core values.yaml

**Priority:** Medium  
**Source:** Architecture: `terminus.infra/apps/crossplane/crossplane-core/values.yaml`  
**Adversarial finding #4:** values.yaml content was undefined

**Context:**  
The architecture lists values.yaml as "Helm values override (if any)". Minimal overrides needed for homelab; this story decides which defaults to accept and which to override.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/crossplane-core/values.yaml`
- [ ] Document rationale for each override (or explicitly note "using upstream defaults")
- [ ] Confirm `revisionHistoryLimit` is set per Agent Consistency Rule 2 (finding #5)
- [ ] File is valid YAML

**Implementation Notes:**
- At minimum: document explicit intent (override or accept default) for: `revisionHistoryLimit`, `metrics.enabled`, `resourcesCrossplane.limits`
- Homelab acceptable: keep defaults unless cluster resources require tuning

---

### Story 1.4: Create kustomization.yaml App-of-Apps Registration

**Priority:** Medium  
**Source:** Architecture: `terminus.infra/apps/crossplane/kustomization.yaml`  
**Adversarial finding #2:** kustomization.yaml content was unspecified

**Context:**  
The kustomization.yaml registers all Crossplane sub-applications with ArgoCD under the App-of-Apps pattern. It must list `crossplane-core`, `providers`, and `providerconfigs` Applications.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/kustomization.yaml`
- [ ] `apiVersion: kustomize.config.k8s.io/v1beta1` and `kind: Kustomization`
- [ ] `resources:` list includes `crossplane-core/Application.yaml`, `providers/`, `providerconfigs/`
- [ ] `namespace: crossplane-system` set
- [ ] File passes `kustomize build .` without errors

---

## Epic 2: Provider Installation

**Goal:** Install and validate `provider-kubernetes` and `provider-helm` via Crossplane Provider CRDs.

**Scope:**
- Author Provider CRD manifests with pinned versions
- Validate InjectedIdentity RBAC requirements for in-cluster operations
- Wire sync-wave ordering to ensure core is healthy before providers install

---

### Story 2.1: Create provider-kubernetes.yaml

**Priority:** High  
**Source:** Architecture: `terminus.infra/apps/crossplane/providers/provider-kubernetes.yaml`  
**Depends-on:** Story 1.1 (version pinned), Story 1.2 (crossplane-core deployed)

**Context:**  
Provider manifest for `upbound/provider-kubernetes`. Uses Crossplane `Provider` CRD kind. Version must be pinned (finding #1 resolved in Story 1.1). Agent Consistency Rule 2: set `revisionHistoryLimit`.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/providers/provider-kubernetes.yaml`
- [ ] `spec.package: xpkg.upbound.io/upbound/provider-kubernetes:vX.Y.Z` (pinned from Story 1.1)
- [ ] `spec.revisionActivationPolicy: Automatic`
- [ ] `spec.revisionHistoryLimit: 1` (homelab — 1 is sufficient)
- [ ] ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "1"` (after core at wave 0)
- [ ] Required labels applied (Agent Consistency Rule 4)
- [ ] File passes `kubectl apply --dry-run=client`

---

### Story 2.2: Create provider-helm.yaml

**Priority:** High  
**Source:** Architecture: `terminus.infra/apps/crossplane/providers/provider-helm.yaml`  
**Depends-on:** Story 1.1, Story 1.2, Story 2.1 (provider pattern established)

**Context:**  
Provider manifest for `upbound/provider-helm`. Identical pattern to provider-kubernetes. Sync wave 1.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/providers/provider-helm.yaml`
- [ ] `spec.package: xpkg.upbound.io/upbound/provider-helm:vX.Y.Z` (pinned from Story 1.1)
- [ ] `spec.revisionActivationPolicy: Automatic`
- [ ] `spec.revisionHistoryLimit: 1`
- [ ] ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "1"`
- [ ] Required labels applied
- [ ] File passes `kubectl apply --dry-run=client`

---

### Story 2.3: Validate InjectedIdentity RBAC for provider-kubernetes

**Priority:** Medium — operational risk (adversarial finding #6)  
**Source:** Adversarial finding #6: InjectedIdentity RBAC requirements unanalyzed  
**Depends-on:** Story 2.1 deployed

**Context:**  
`InjectedIdentity` means the provider pod uses the Crossplane ServiceAccount (SA) for in-cluster operations. For `provider-kubernetes`, the SA may need specific ClusterRoles to create, update, and delete k8s objects across namespaces. This is a known operational gap that must be resolved before managed resources can function.

**Acceptance Criteria:**
- [ ] Research Crossplane `provider-kubernetes` RBAC requirements for InjectedIdentity
- [ ] Verify the default Crossplane ServiceAccount has required ClusterRole bindings
- [ ] If additional RBAC needed: create `rbac-provider-kubernetes.yaml` in `crossplane-core/`
- [ ] Document RBAC posture decision in story notes
- [ ] Smoke test: create a test `Object` managed resource and verify it reconciles successfully

**Implementation Notes:**
- Crossplane installs a `crossplane` ClusterRole and SA by default
- `provider-kubernetes` with InjectedIdentity inherits this SA — check if it covers cross-namespace operations
- Homelab context: RBAC hardening is deferred; broadest-needed permissions acceptable

---

## Epic 3: ProviderConfig, Sync Orchestration, and Acceptance

**Goal:** Wire ProviderConfigs, establish sync-wave ordering, and validate end-to-end installation.

**Scope:**
- Create ProviderConfig manifests for both providers
- Configure ArgoCD sync wave ordering (core → providers → providerconfigs)
- Define and execute post-install acceptance criteria (adversarial finding #10)

---

### Story 3.1: Create kubernetes-default ProviderConfig

**Priority:** High  
**Source:** Architecture: `terminus.infra/apps/crossplane/providerconfigs/kubernetes-default.yaml`  
**Depends-on:** Story 2.1 (provider-kubernetes deployed and HEALTHY)

**Context:**  
ProviderConfig sets `spec.credentials.source: InjectedIdentity` for the `provider-kubernetes`. Referenced by all future managed resources via `providerConfigRef.name: default`.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/providerconfigs/kubernetes-default.yaml`
- [ ] `kind: ProviderConfig`, `apiVersion: kubernetes.crossplane.io/v1alpha1`
- [ ] `metadata.name: default`
- [ ] `spec.credentials.source: InjectedIdentity`
- [ ] ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "2"` (after providers at wave 1)
- [ ] Required labels applied
- [ ] File passes `kubectl apply --dry-run=client`

---

### Story 3.2: Create helm-default ProviderConfig

**Priority:** High  
**Source:** Architecture: `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml`  
**Depends-on:** Story 2.2 (provider-helm deployed and HEALTHY)

**Context:**  
ProviderConfig for `provider-helm`. Identical credential strategy (InjectedIdentity). Referenced by future `HelmRelease` managed resources.

**Acceptance Criteria:**
- [ ] Create `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml`
- [ ] `kind: ProviderConfig`, `apiVersion: helm.crossplane.io/v1beta1`
- [ ] `metadata.name: default`
- [ ] `spec.credentials.source: InjectedIdentity`
- [ ] ArgoCD sync wave annotation: `argocd.argoproj.io/sync-wave: "2"`
- [ ] Required labels applied
- [ ] File passes `kubectl apply --dry-run=client`

---

### Story 3.3: Configure ArgoCD Sync Wave Ordering

**Priority:** High — deployment correctness  
**Source:** Adversarial finding #9: sync wave ordering omitted  
**Depends-on:** Stories 1.2, 2.1, 2.2, 3.1, 3.2

**Context:**  
Crossplane requires strict installation ordering: core must be healthy before providers install (CRDs must exist), providers must be healthy before ProviderConfigs apply (ProviderConfig CRDs must exist). Incorrect ordering causes ArgoCD sync failures. This story validates and unifies sync-wave annotations across all manifests.

**Acceptance Criteria:**
- [ ] Verify sync-wave annotations: `crossplane-core` = wave 0, `providers` = wave 1, `providerconfigs` = wave 2
- [ ] ArgoCD Application for crossplane has `spec.syncPolicy.syncOptions: [ServerSideApply=true]` (needed for CRD ownership)
- [ ] Validate wave ordering in a staging ArgoCD Application before prod commit
- [ ] `selfHeal: true` and `prune: true` confirmed on all Applications (finding #11)
- [ ] Document final sync-wave configuration in `docs/terminus/infra/crossplane/architecture.md` under "Deferred Decisions"

---

### Story 3.4: Post-Install Acceptance Testing

**Priority:** High — done criteria  
**Source:** Adversarial finding #10: no acceptance criteria defined  
**Depends-on:** All prior stories complete

**Context:**  
Without defined acceptance criteria there are no done criteria for implementation. This story defines and executes the post-install validation sequence.

**Acceptance Criteria:**
- [ ] Crossplane pods running: `kubectl get pods -n crossplane-system` → all Running
- [ ] Provider pods healthy: `kubectl get providers` → `HEALTHY=True` for both providers
- [ ] ProviderConfig status: `kubectl get providerconfigs` → both show `SYNCED=True`
- [ ] CRDs registered: `kubectl get crds | grep crossplane.io` → Provider, ProviderConfig, Object, Release CRDs present
- [ ] Smoke test: apply a test `Object` managed resource targeting default namespace
- [ ] ArgoCD Application status: all 3 Applications `Synced` and `Healthy`
- [ ] Document actual post-install state in `docs/terminus/infra/crossplane/acceptance-results.md`

---

## Story Dependency Map

```
1.1 (version pin) → 1.2 (App) → 1.3 (values) → 1.4 (kustomization)
                              ↓
                   2.1 (provider-k8s) → 2.3 (RBAC validation) → 3.1 (k8s ProviderConfig)
                   2.2 (provider-helm)                         → 3.2 (helm ProviderConfig)
                              ↓
                   3.3 (sync waves) → 3.4 (acceptance)
```

**Safe sequential order for single developer:**  
`1.1 → 1.2 → 1.3 → 1.4 → 2.1 → 2.2 → 2.3 → 3.1 → 3.2 → 3.3 → 3.4`

---

## Open Items from Adversarial Review (carried to stories)

| Finding | Story | Resolution |
|---|---|---|
| #1 Provider versions not pinned | Story 1.1 | Must pin before any implementation |
| #2 kustomization.yaml unspecified | Story 1.4 | Full spec in story |
| #3 ArgoCD Application spec absent | Story 1.2 | Full spec in story |
| #4 values.yaml content undefined | Story 1.3 | Explicit decision in story |
| #5 revisionHistoryLimit unstated | Stories 2.1, 2.2 | Set to 1 |
| #6 InjectedIdentity RBAC unanalyzed | Story 2.3 | Dedicated validation story |
| #7 No environment matrix | Not scoped | Phase 1 = single homelab env; deferred |
| #8 No rollback strategy | Story 3.3 | ArgoCD prune/selfHeal provides recovery |
| #9 Sync wave ordering omitted | Story 3.3 | Dedicated ordering story |
| #10 No acceptance criteria | Story 3.4 | Dedicated validation story |
| #11 Self-heal/prune not specified | Story 1.2, 3.3 | Explicit in Application spec |
| #12 terminus.io/managed-by label | All stories | Accepted as convention — tracked |
