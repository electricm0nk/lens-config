# Story 2.2: Create provider-helm.yaml

Status: ready-for-dev

## Story

As a platform engineer,
I want a Crossplane `Provider` CRD manifest for `upbound/provider-helm` with a pinned version and correct sync-wave annotation,
so that ArgoCD installs the Helm provider deterministically after `crossplane-core` is healthy.

## Context

`provider-helm` enables management of Helm releases (`Release` CRDs) via Crossplane.
It follows the exact same pattern as `provider-kubernetes` and lives in the same `providers/` directory.

**Depends-on:** Story 1.1 (version pinned — use `v1.2.1`), Story 2.1 (providers/Application.yaml already created)  
**Target file:** `terminus.infra/apps/crossplane/providers/provider-helm.yaml`

### Pinned Version (from Story 1.1)

```
xpkg.upbound.io/upbound/provider-helm:v1.2.1
```
> Verify version is still current before implementing. Source: https://marketplace.upbound.io/providers/upbound/provider-helm

### Adversarial Findings Addressed

- **Finding #1:** Provider version pinned (`v1.2.1`)
- **Finding #5:** `revisionHistoryLimit: 1`
- **Finding #9:** Sync wave `"1"` — same wave as provider-kubernetes

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/providers/provider-helm.yaml` exists and is valid YAML
2. `spec.package: xpkg.upbound.io/upbound/provider-helm:v1.2.1` (or newer pinned from Story 1.1)
3. `spec.revisionActivationPolicy: Automatic`
4. `spec.revisionHistoryLimit: 1`
5. ArgoCD sync-wave annotation: `argocd.argoproj.io/sync-wave: "1"`
6. Required labels: `app.kubernetes.io/part-of: crossplane` and `terminus.io/managed-by: argocd`
7. File passes `kubectl apply --dry-run=client -f provider-helm.yaml`
8. `providers/Application.yaml` from Story 2.1 is NOT modified — it already covers both providers

## Tasks / Subtasks

- [ ] Task 1: Write provider-helm.yaml (AC: #1–#6)
  - [ ] Use the template in Dev Notes
  - [ ] Confirm version matches Story 1.1 resolution

- [ ] Task 2: Validate (AC: #7)
  - [ ] `kubectl apply --dry-run=client -f provider-helm.yaml`
  - [ ] Confirm `providers/Application.yaml` from Story 2.1 still looks correct with both files present

## Dev Notes

### Target Repository

All files go into **`terminus.infra`** repo.
Path: `terminus.infra/apps/crossplane/providers/provider-helm.yaml`

### Manifest Template

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-helm
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  package: xpkg.upbound.io/upbound/provider-helm:v1.2.1
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

### InjectedIdentity for provider-helm

Unlike `provider-kubernetes`, `provider-helm` does NOT require an additional ClusterRoleBinding
for InjectedIdentity. The Helm provider operates within the cluster's existing permissions
for deploying releases into namespaces. The default Crossplane ServiceAccount is sufficient.

This was confirmed via marketplace docs review in Story 1.1. Document if smoke testing
in Story 2.3 reveals otherwise.

### Pattern Consistency

This manifest is intentionally identical in structure to `provider-kubernetes.yaml` (Story 2.1)
except for the `metadata.name` and `spec.package` fields. This is correct — maintain the pattern.
Do not introduce any differences unless there is a specific functional reason.

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Provider Selection, Consistency Rules]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Findings #1, #5, #9]
- [Source: docs/implementation-artifacts/1-1-research-and-pin-crossplane-provider-versions.md]
- [Upstream: https://marketplace.upbound.io/providers/upbound/provider-helm/v1.2.1]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/providers/provider-helm.yaml` — create
