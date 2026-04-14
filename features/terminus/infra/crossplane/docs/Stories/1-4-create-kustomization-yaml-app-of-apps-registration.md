# Story 1.4: Create kustomization.yaml App-of-Apps Registration

Status: ready-for-dev

## Story

As a platform engineer,
I want a `kustomization.yaml` at the root of the crossplane app directory that registers all crossplane sub-applications with ArgoCD,
so that a single ArgoCD Application pointing at the root kustomization boots the entire Crossplane platform stack.

## Context

The App-of-Apps pattern requires a root kustomization that lists all child Applications.
ArgoCD watches this kustomization and syncs all referenced Applications automatically.
Without it, the crossplane-core, providers, and providerconfigs Applications are orphaned.

**Adversarial finding #2:** kustomization.yaml content was unspecified — no resource entries,
namespace, or apiVersion were defined. This story fills that gap.

**Target file:** `terminus.infra/apps/crossplane/kustomization.yaml`

**Depends-on:** Stories 1.2 (crossplane-core/Application.yaml exists to reference)

> Note: This file can be created and committed before Stories 2.1–3.2 complete their Application.yaml
> files. ArgoCD will simply report those as missing until the referenced files exist.
> The kustomization should reference the future Application paths even if not yet created.

## Acceptance Criteria

1. File `terminus.infra/apps/crossplane/kustomization.yaml` exists and is valid YAML
2. `apiVersion: kustomize.config.k8s.io/v1beta1` and `kind: Kustomization` are present
3. `resources:` list includes all sub-application paths:
   - `crossplane-core/Application.yaml`
   - `providers/Application.yaml`
   - `providerconfigs/Application.yaml`
4. File passes `kustomize build terminus.infra/apps/crossplane/` without errors (only after Application.yaml files exist)
5. No inline namespace override at kustomization level (each Application manages its own namespace)

## Tasks / Subtasks

- [ ] Task 1: Create kustomization.yaml (AC: #1–#4)
  - [ ] Use the template in Dev Notes
  - [ ] Verify all three resource paths are correct relative to this file's location

- [ ] Task 2: Validate with kustomize (AC: #4)
  - [ ] `kustomize build terminus.infra/apps/crossplane/` — expect 3 Application manifests output
  - [ ] Validate only after Stories 2.1 and 3.1 have created their Application.yaml files

## Dev Notes

### Target Repository

All files go into the **`terminus.infra`** repo.
Path: `terminus.infra/apps/crossplane/kustomization.yaml`

### Full Directory Layout After All Stories Complete

```
terminus.infra/apps/crossplane/
├── kustomization.yaml                        ← this story
├── crossplane-core/
│   ├── Application.yaml                      ← Story 1.2
│   └── values.yaml                           ← Story 1.3
├── providers/
│   ├── Application.yaml                      ← Story 2.1 (see Dev Notes)
│   ├── provider-kubernetes.yaml              ← Story 2.1
│   └── provider-helm.yaml                    ← Story 2.2
└── providerconfigs/
    ├── Application.yaml                      ← Story 3.1 (see Dev Notes)
    ├── kubernetes-default.yaml               ← Story 3.1
    └── helm-default.yaml                     ← Story 3.2
```

### Note on providers/ and providerconfigs/ Application.yaml

The arch document structure has Provider CRD files directly in the providers/ directory.
Stories 2.1 and 2.2 create Provider CRD manifests. The kustomization references an
`Application.yaml` per sub-directory that wraps those CRDs in an ArgoCD Application.

Stories 2.1 and 3.1 need to also create an ArgoCD `Application.yaml` in their directories
that points ArgoCD at the directory path. This kustomization references those Application files
(not the raw Provider/ProviderConfig CRD files directly).

### Manifest Template

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Crossplane App-of-Apps root
# Registers all Crossplane sub-applications with ArgoCD
# ArgoCD watches this kustomization via the root crossplane Application

resources:
  - crossplane-core/Application.yaml
  - providers/Application.yaml
  - providerconfigs/Application.yaml
```

### Why No Namespace Override

Each ArgoCD Application specifies its own target namespace (`argocd` for the Application resource itself,
`crossplane-system` for the destination). Adding a kustomization-level namespace override would
incorrectly remap the Application resources. Leave namespace management to each Application.

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — Project Structure, App-of-Apps pattern]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Finding #2]
- [Upstream: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- `terminus.infra/apps/crossplane/kustomization.yaml` — create
