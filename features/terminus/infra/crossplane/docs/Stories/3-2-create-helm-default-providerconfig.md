# Story 3.2: Create helm-default ProviderConfig

Status: ready-for-dev

## Story

As a platform engineer,
I want a `ProviderConfig` manifest for `provider-helm` using `InjectedIdentity` credentials named `default`,
so that all future Crossplane `Release` managed resources can reference `providerConfigRef.name: default` without additional configuration.

## Context

Mirrors Story 3.1 exactly but for `provider-helm`. The credential strategy is the same
(`InjectedIdentity`), the naming convention is the same (`default`), and both ProviderConfigs
live in the same `providerconfigs/` directory.

**Depends-on:** Story 2.2 (provider-helm deployed, preferably HEALTHY), Story 3.1 (providerconfigs/Application.yaml already created)  
**Target file:** `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml`

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml` exists and is valid YAML
2. `apiVersion: helm.crossplane.io/v1beta1`
3. `kind: ProviderConfig`
4. `metadata.name: default`
5. `spec.credentials.source: InjectedIdentity`
6. ArgoCD sync-wave annotation: `argocd.argoproj.io/sync-wave: "2"`
7. Required labels: `app.kubernetes.io/part-of: crossplane` and `terminus.io/managed-by: argocd`
8. `providerconfigs/Application.yaml` from Story 3.1 is NOT modified — it already covers both ProviderConfigs
9. File passes `kubectl apply --dry-run=client -f helm-default.yaml`

## Tasks / Subtasks

- [ ] Task 1: Write helm-default.yaml (AC: #1–#7)
  - [ ] Use the template in Dev Notes
  - [ ] Verify apiVersion is correct for provider-helm (differs from provider-kubernetes)

- [ ] Task 2: Validate (AC: #9)
  - [ ] `kubectl apply --dry-run=client -f helm-default.yaml`

## Dev Notes

### Target Repository

All files go into **`terminus.infra`** repo.
Path: `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml`

### helm-default.yaml Template

```yaml
apiVersion: helm.crossplane.io/v1beta1
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

### Critical: apiVersion Difference from Story 3.1

| Provider | ProviderConfig apiVersion |
|---|---|
| provider-kubernetes | `kubernetes.crossplane.io/v1alpha1` |
| provider-helm | `helm.crossplane.io/v1beta1` |

These are different CRD groups installed by each respective provider. Using the wrong apiVersion
will cause ArgoCD sync failure (`no matches for kind "ProviderConfig" in version ...`).
Check installed CRDs after provider deployment: `kubectl get crds | grep crossplane.io`

### Both ProviderConfigs Named `default`

Having both `kubernetes-default.yaml` and `helm-default.yaml` each declaring `metadata.name: default`
is correct and expected. They are in different CRD groups and do not conflict:
- `providerconfigs.kubernetes.crossplane.io/default`
- `providerconfigs.helm.crossplane.io/default`

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Consistency Rule 3, Credential Strategy table]
- [Upstream: https://marketplace.upbound.io/providers/upbound/provider-helm/v1.2.1 — ProviderConfig examples]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/providerconfigs/helm-default.yaml` — create
