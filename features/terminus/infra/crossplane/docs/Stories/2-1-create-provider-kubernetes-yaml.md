# Story 2.1: Create provider-kubernetes.yaml

Status: ready-for-dev

## Story

As a platform engineer,
I want a Crossplane `Provider` CRD manifest for `upbound/provider-kubernetes` with a pinned version and correct sync-wave annotation,
so that ArgoCD installs the provider deterministically after `crossplane-core` is healthy.

## Context

`provider-kubernetes` enables management of arbitrary Kubernetes objects (`Object` CRDs)
via Crossplane. It is the primary provider for in-cluster resource management in terminus.

**Depends-on:** Story 1.1 (version pinned — use `v1.2.2`), Story 1.2 (crossplane-core must be deployed first)  
**Target file:** `terminus.infra/apps/crossplane/providers/provider-kubernetes.yaml`

Additionally, this story must create an ArgoCD `Application.yaml` that wraps the providers/
directory so that ArgoCD syncs Provider manifests. The kustomization (Story 1.4) references
`providers/Application.yaml`.

### Pinned Version (from Story 1.1)

```
xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2
```
> Verify version is still current before implementing. Source: https://marketplace.upbound.io/providers/upbound/provider-kubernetes

### Adversarial Findings Addressed

- **Finding #1:** Provider version is now pinned (`v1.2.2`) — not `latest`
- **Finding #5:** `revisionHistoryLimit` is set to `1`
- **Finding #9:** Sync wave `"1"` — ensures core (wave 0) is healthy before providers install

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/providers/provider-kubernetes.yaml` exists and is valid YAML
2. `spec.package: xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2` (or newer pinned from Story 1.1)
3. `spec.revisionActivationPolicy: Automatic`
4. `spec.revisionHistoryLimit: 1`
5. ArgoCD sync-wave annotation: `argocd.argoproj.io/sync-wave: "1"` on metadata.annotations
6. Required labels: `app.kubernetes.io/part-of: crossplane` and `terminus.io/managed-by: argocd`
7. File `terminus.infra/apps/crossplane/providers/Application.yaml` exists — argoCD Application pointing at the providers/ path
8. Application.yaml has `argocd.argoproj.io/sync-wave: "1"` and `syncPolicy.automated: {prune: true, selfHeal: true}`
9. Files pass `kubectl apply --dry-run=client`

## Tasks / Subtasks

- [ ] Task 1: Create providers/ directory and write provider-kubernetes.yaml (AC: #1–#6)
  - [ ] `mkdir -p terminus.infra/apps/crossplane/providers/`
  - [ ] Write manifest using the template in Dev Notes
  - [ ] Confirm version matches Story 1.1 resolution

- [ ] Task 2: Create providers/Application.yaml (AC: #7–#8)
  - [ ] ArgoCD Application that points at `terminus.infra/apps/crossplane/providers/` path
  - [ ] Sync wave `"1"`, automated prune + selfHeal

- [ ] Task 3: Validate manifests (AC: #9)
  - [ ] `kubectl apply --dry-run=client -f provider-kubernetes.yaml`

## Dev Notes

### Target Repository

All files go into **`terminus.infra`** repo.

### provider-kubernetes.yaml Template

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  package: xpkg.upbound.io/upbound/provider-kubernetes:v1.2.2
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

### providers/Application.yaml Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-providers
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    repoURL: <terminus.infra-repo-url>
    targetRevision: HEAD
    path: apps/crossplane/providers
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

### RBAC Reminder

After deploying `provider-kubernetes`, Story 2.3 must run the ClusterRoleBinding
before ProviderConfigs can reconcile `Object` resources. Do NOT skip Story 2.3.

```bash
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
  sed -e 's|serviceaccount\/|crossplane-system:|g')
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount="${SA}"
```

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Provider Selection, Pinned Versions, Agent Consistency Rules]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Findings #1, #5, #9]
- [Source: docs/implementation-artifacts/1-1-research-and-pin-crossplane-provider-versions.md]
- [Upstream: https://marketplace.upbound.io/providers/upbound/provider-kubernetes/v1.2.2]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/providers/provider-kubernetes.yaml` — create
- `terminus.infra/apps/crossplane/providers/Application.yaml` — create
