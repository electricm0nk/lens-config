# Story 1.1: Add sync-wave annotations to critical ordering gaps

## Status: ready-for-dev

## Story

As a platform engineer,
I want `argocd.argoproj.io/sync-wave` annotations added to the 7 ArgoCD Application objects that currently have no wave annotation,
So that `temporal-infra`, all `fourdogs-*-infra` namespace-prep apps, `terminus-portal-infra`, and `semaphoreui` no longer race with cluster primitives at wave 0 and instead load at their correct dependency tier.

## Acceptance Criteria

- **Given** `platforms/k3s/argocd/apps/temporal-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added under `metadata.annotations`
- **Then** ArgoCD deploys temporal-infra after all wave-0 primitives are healthy

- **Given** `platforms/k3s/argocd/apps/fourdogs-central-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added
- **Then** fourdogs-central-infra deploys at wave 1

- **Given** `platforms/k3s/argocd/apps/fourdogs-central-dev-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added
- **Then** fourdogs-central-dev-infra deploys at wave 1

- **Given** `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added
- **Then** fourdogs-kaylee-agent-infra deploys at wave 1

- **Given** `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-dev-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added
- **Then** fourdogs-kaylee-agent-dev-infra deploys at wave 1

- **Given** `platforms/k3s/argocd/apps/terminus-portal-infra.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "1"` is added
- **Then** terminus-portal-infra deploys at wave 1

- **Given** `platforms/k3s/argocd/apps/semaphoreui.yaml` has no sync-wave annotation
- **When** `argocd.argoproj.io/sync-wave: "5"` is added
- **Then** semaphoreui deploys after wave-4 actions-runner, before wave-6 workloads

- **Given** all 7 annotations are applied and ArgoCD syncs `terminus-infra-k3s-root`
- **When** the following command is run:
  ```bash
  kubectl get applications -n argocd \
    -o custom-columns=\
  'NAME:.metadata.name,WAVE:.metadata.annotations.argocd\.argoproj\.io/sync-wave' \
    --sort-by='.metadata.annotations.argocd\.argoproj\.io/sync-wave'
  ```
- **Then** `temporal-infra`, `fourdogs-central-infra`, `fourdogs-central-dev-infra`, `fourdogs-kaylee-agent-infra`, `fourdogs-kaylee-agent-dev-infra`, `terminus-portal-infra` all show wave `1`
- **And** `semaphoreui` shows wave `5`

## Tasks / Subtasks

- [ ] Task 1: Edit `platforms/k3s/argocd/apps/temporal-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations` (create `annotations:` block if absent)
- [ ] Task 2: Edit `platforms/k3s/argocd/apps/fourdogs-central-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations`
- [ ] Task 3: Edit `platforms/k3s/argocd/apps/fourdogs-central-dev-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations`
- [ ] Task 4: Edit `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations`
- [ ] Task 5: Edit `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-dev-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations`
- [ ] Task 6: Edit `platforms/k3s/argocd/apps/terminus-portal-infra.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "1"` under `metadata.annotations`
- [ ] Task 7: Edit `platforms/k3s/argocd/apps/semaphoreui.yaml`
  - [ ] Add `argocd.argoproj.io/sync-wave: "5"` under `metadata.annotations`
- [ ] Task 8: Commit all 7 files
  - [ ] Commit message: `fix(argocd): add sync-wave annotations to infra apps and semaphoreui`
- [ ] Task 9: After ArgoCD sync, run wave-compliance verification command and confirm output

## Dev Notes

**Annotation format:** ArgoCD sync-wave values must be quoted strings, not bare integers:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

**`annotations:` block may not exist:** Some of these Application objects may have only `labels:` under `metadata:` or no `annotations:` key at all. Add `annotations:` as a sibling to `labels:` if absent.

**Why wave 1 for infra apps:** These apps provision namespaces and ExternalSecret CRs. If they race at wave 0 with ESO (which also has no annotation and is effectively wave 0), ESO may not be ready when the ExternalSecret resources are created, causing them to fail reconciliation that then blocks workload apps from starting.

**Why wave 5 for semaphoreui:** SemaphoreUI uses inline secrets (not ExternalSecret-sourced) and has no dependency on temporal-workers. Wave 5 places it after the CI runner tier (wave 4) and before general workloads (wave 6).

**Rollback:** `git revert <commit-sha>` — cluster state is unaffected since sync-wave annotations are ArgoCD-side metadata only.

**Reference:** `docs/terminus/infra/k3s-startup-sequence/architecture.md` — wave map and rationale.

## Dev Agent Record

### Agent Model Used: ``
### Completion Notes List

### Change List
- `platforms/k3s/argocd/apps/temporal-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/fourdogs-central-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/fourdogs-central-dev-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/fourdogs-kaylee-agent-dev-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/terminus-portal-infra.yaml` — add annotation
- `platforms/k3s/argocd/apps/semaphoreui.yaml` — add annotation
