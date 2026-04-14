# Story 3.1: Create kubernetes-default ProviderConfig

Status: ready-for-dev

## Story

As a platform engineer,
I want a `ProviderConfig` manifest for `provider-kubernetes` using `InjectedIdentity` credentials named `default`,
so that all future Crossplane `Object` managed resources can reference `providerConfigRef.name: default` without additional configuration.

## Context

The ProviderConfig is the credential binding that connects a Provider to a credential source.
For `InjectedIdentity`, it simply declares that the provider uses the in-cluster ServiceAccount.
All managed resources in terminus will reference this ProviderConfig by the name `default`
(Agent Consistency Rule 3: always use `name: default` for single-cluster deployments).

**Depends-on:** Story 2.1 (provider-kubernetes deployed, preferably HEALTHY)  
**Target file:** `terminus.infra/apps/crossplane/providerconfigs/kubernetes-default.yaml`

Additionally this story must create an ArgoCD `Application.yaml` for the `providerconfigs/`
directory (referenced by the kustomization in Story 1.4).

### Adversarial Finding Addressed

- **Finding #9:** Sync wave `"2"` — ProviderConfigs must wait until providers (wave 1) are healthy

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/providerconfigs/kubernetes-default.yaml` exists and is valid YAML
2. `apiVersion: kubernetes.crossplane.io/v1alpha1`
3. `kind: ProviderConfig`
4. `metadata.name: default`
5. `spec.credentials.source: InjectedIdentity`
6. ArgoCD sync-wave annotation: `argocd.argoproj.io/sync-wave: "2"` on metadata.annotations
7. Required labels: `app.kubernetes.io/part-of: crossplane` and `terminus.io/managed-by: argocd`
8. File `terminus.infra/apps/crossplane/providerconfigs/Application.yaml` exists — ArgoCD Application pointing at providerconfigs/ path, sync-wave `"2"`
9. Files pass `kubectl apply --dry-run=client`

## Tasks / Subtasks

- [ ] Task 1: Create providerconfigs/ directory and write kubernetes-default.yaml (AC: #1–#7)
  - [ ] `mkdir -p terminus.infra/apps/crossplane/providerconfigs/`
  - [ ] Write manifest using the template in Dev Notes

- [ ] Task 2: Create providerconfigs/Application.yaml (AC: #8)
  - [ ] ArgoCD Application pointing at `terminus.infra/apps/crossplane/providerconfigs/` path
  - [ ] Sync wave `"2"`, automated prune + selfHeal

- [ ] Task 3: Validate (AC: #9)
  - [ ] `kubectl apply --dry-run=client -f kubernetes-default.yaml`

## Dev Notes

### Target Repository

All files go into **`terminus.infra`** repo.

### kubernetes-default.yaml Template

```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  credentials:
    source: InjectedIdentity
```

### providerconfigs/Application.yaml Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-providerconfigs
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: <terminus.infra-repo-url>
    targetRevision: HEAD
    path: apps/crossplane/providerconfigs
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

### Architecture Consistency Rule 3

> All managed resources reference `providerConfigRef.name: default` unless explicitly multi-cluster.

The name `default` is not arbitrary — it is the convention that ALL future terminus initiatives
creating `Object` managed resources will use. Do not rename it.

### apiVersion Note

```
kubernetes.crossplane.io/v1alpha1
```

This is the correct apiVersion for the `provider-kubernetes` ProviderConfig. It differs from
the Crossplane core apiVersion (`pkg.crossplane.io/v1`). Check the provider CRD list after
deployment: `kubectl get crds | grep kubernetes.crossplane.io`

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Consistency Rule 3, Project Structure]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Finding #9]
- [Upstream: https://marketplace.upbound.io/providers/upbound/provider-kubernetes/v1.2.2 — ProviderConfig examples]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/providerconfigs/kubernetes-default.yaml` — create
- `terminus.infra/apps/crossplane/providerconfigs/Application.yaml` — create
