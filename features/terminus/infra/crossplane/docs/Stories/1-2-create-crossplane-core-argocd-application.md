# Story 1.2: Create crossplane-core ArgoCD Application

Status: ready-for-dev

## Story

As a platform engineer,
I want an ArgoCD Application manifest that deploys the Crossplane Helm chart into the k3s cluster,
so that Crossplane is installed and managed declaratively via the App-of-Apps GitOps pattern.

## Context

This story creates the primary delivery mechanism for Crossplane. The ArgoCD Application
points at the upstream Crossplane Helm chart (crossplane-stable registry) and installs it
into the `crossplane-system` namespace. It is the foundation that all other stories depend on.

**Depends-on:** Story 1.1 (provider versions pinned — Crossplane chart version must also be verified)

**Target file:** `terminus.infra/apps/crossplane/crossplane-core/Application.yaml`

### What architecture.md specifies

From [architecture.md](../../docs/terminus/infra/crossplane/architecture.md):
- **Starter template:** Official Crossplane Helm chart (`crossplane-stable/crossplane`) in an ArgoCD `Application`
- **Delivery pattern:** ArgoCD App-of-Apps: `terminus.infra/apps/crossplane/crossplane-core/Application.yaml`
- **Namespace:** `crossplane-system`
- **Helm registry:** `https://charts.crossplane.io/stable`
- **No UXP** — official chart only

### Adversarial Findings Addressed

- **Finding #3:** ArgoCD Application spec was entirely absent — this story provides it in full
- **Finding #9:** Sync wave ordering — crossplane-core must be wave `"0"` (providers are wave `"1"`, providerconfigs wave `"2"`)
- **Finding #11:** `selfHeal: true` and `prune: true` must be explicitly set

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/crossplane-core/Application.yaml` exists and is valid YAML
2. `spec.source.repoURL: https://charts.crossplane.io/stable`
3. `spec.source.chart: crossplane` with pinned `targetRevision` (latest stable from https://artifacthub.io/packages/helm/crossplane/crossplane)
4. `spec.destination.namespace: crossplane-system`
5. `spec.destination.server: https://kubernetes.default.svc` (in-cluster)
6. `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
7. `spec.syncPolicy.syncOptions` includes `ServerSideApply=true` (required for CRD ownership)
8. `spec.syncPolicy.syncOptions` includes `CreateNamespace=true` (crossplane-system may not exist yet)
9. ArgoCD sync-wave annotation: `argocd.argoproj.io/sync-wave: "0"` on the Application metadata
10. Required labels present: `app.kubernetes.io/part-of: crossplane` and `terminus.io/managed-by: argocd`
11. File passes `kubectl apply --dry-run=client -f Application.yaml` against target cluster

## Tasks / Subtasks

- [ ] Task 1: Verify current crossplane Helm chart version (AC: #3)
  - [ ] Check https://artifacthub.io/packages/helm/crossplane/crossplane for latest stable `targetRevision`
  - [ ] Confirm chart version ≥ 2.0.0 (required for provider-kubernetes v1.2.2 and provider-helm v1.2.1 compat)

- [ ] Task 2: Create directory structure in terminus.infra (AC: #1)
  - [ ] `mkdir -p terminus.infra/apps/crossplane/crossplane-core/`

- [ ] Task 3: Write Application.yaml (AC: #1–#10)
  - [ ] Use the template in Dev Notes below as the starting point
  - [ ] Fill in `targetRevision` from Task 1
  - [ ] Verify all fields are correct

- [ ] Task 4: Validate manifest (AC: #11)
  - [ ] `kubectl apply --dry-run=client -f terminus.infra/apps/crossplane/crossplane-core/Application.yaml`

## Dev Notes

### Target Repository

All files created in this story go into the **`terminus.infra`** repo, NOT the control repo.
Path: `TargetProjects/terminus/infra/terminus.infra/` (local path — verify against `initiative.target_repos[0].local_path`)

### Manifest Template

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-core
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: crossplane
    terminus.io/managed-by: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://charts.crossplane.io/stable
    chart: crossplane
    targetRevision: <PIN-VERSION-FROM-TASK-1>
    helm:
      valuesFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Sync Wave Ordering — Critical

The full 3-wave ordering across all stories:
- **Wave "0":** `crossplane-core` Application (this story) — installs CRDs
- **Wave "1":** `crossplane-providers` Application (Story 2.1/2.2) — Provider objects
- **Wave "2":** `crossplane-providerconfigs` Application (Stories 3.1/3.2) — ProviderConfig objects

ArgoCD will not advance to a higher wave until all lower-wave resources are `Healthy`.
This is the mechanism that ensures: core is healthy → providers install → providerconfigs apply.

### values.yaml Relationship

Story 1.3 creates `values.yaml` in the same directory. The `helm.valuesFiles: [values.yaml]`
reference in this Application manifest will resolve against the Application's source path.
If values.yaml doesn't exist yet (Story 1.3 not done), ArgoCD will warn but not fail on an empty file.

### Notes on `ServerSideApply=true`

Crossplane installs cluster-scoped CRDs. ArgoCD requires `ServerSideApply=true` to correctly
manage field ownership on these CRDs. Without it, sync conflicts will occur on CRD fields.
This is a known Crossplane + ArgoCD integration requirement.

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Starter Template, Project Structure]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Findings #3, #9, #11]
- [Upstream: https://artifacthub.io/packages/helm/crossplane/crossplane]
- [Upstream: https://docs.crossplane.io/latest/software/install/]
- [Upstream: https://docs.crossplane.io/latest/guides/git-ops/]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/crossplane-core/Application.yaml` — create
