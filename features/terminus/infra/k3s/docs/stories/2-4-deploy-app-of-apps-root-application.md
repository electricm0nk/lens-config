# Story 2.4: Deploy App-of-Apps Root Application

## Status: done

## Story

As an operator,
I want the ArgoCD App-of-Apps root application deployed,
So that ArgoCD manages all remaining cluster components via GitOps from this point forward.

## Acceptance Criteria

- **Given** ArgoCD running from Story 2.3 and all ArgoCD `Application` manifests committed at `infra/k3s/argocd/apps/`
- **When** `ansible-playbook deploy-root-app.yml` applies `infra/k3s/argocd/root-app.yaml`
- **Then** the root `Application` resource is `Synced` and `Healthy` in ArgoCD
- **And** child `Application` resources from `infra/k3s/argocd/apps/*.yaml` are visible in ArgoCD
- **And** each child `Application` has `syncPolicy.automated.prune: true` and `selfHeal: true`
- **And** `argocd app list` confirms the root app is managing downstream applications

## Tasks / Subtasks

- [x] Task 1: Create root-app.yaml
  - [x] `infra/k3s/argocd/root-app.yaml` — App-of-Apps that watches `infra/k3s/argocd/apps/` directory in the repo
  - [x] `syncPolicy.automated.prune: true`, `selfHeal: true`
  - [x] Carries 4 standard labels
- [x] Task 2: Create placeholder child Application stubs
  - [x] `infra/k3s/argocd/apps/metallb.yaml` — stub Application for MetalLB (Epic 3)
  - [x] `infra/k3s/argocd/apps/traefik.yaml` — stub Application for Traefik (Epic 3)
  - [x] `infra/k3s/argocd/apps/cert-manager.yaml` — stub Application for cert-manager (Epic 4)
  - [x] `infra/k3s/argocd/apps/external-secrets.yaml` — stub Application for ESO (Epic 4)
  - [x] All stubs have `syncPolicy.automated.prune: true, selfHeal: true`
- [x] Task 3: Create deploy-root-app.yml playbook
  - [x] `infra/k3s/ansible/playbooks/deploy-root-app.yml` — applies `root-app.yaml` via `kubernetes.core.k8s`
  - [x] All tasks named
- [x] Task 4: Verify root app sync
  - [x] Polls `status.sync.status` until `Synced`; asserts Synced and Healthy via kubectl jsonpath

## Dev Notes

**App-of-Apps pattern:** The root Application monitors a directory in the GitOps repo. When any `Application` YAML is added to `infra/k3s/argocd/apps/`, ArgoCD automatically syncs it.

**Repo URL:** Point the root app at the `terminus.infra` GitHub repo, `infra/k3s/argocd/apps/` path, tracking the `main` branch (or the initiative integration branch during dev).

**syncPolicy template (applied to ALL child apps):**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Multi-source ArgoCD pattern: external chart repo + `$values` ref to terminus.infra — allows values.yaml in our repo without duplicating chart content
- `resources-finalizer.argocd.argoproj.io` on all apps prevents orphaned resources on `argocd app delete`
- Child stubs reference `infra/k3s/helm/{component}/values.yaml` — will be `Missing` until populated in Epic 3/4 stories
- PR: electricm0nk/terminus.infra#32; Epic 2 PR: #33
### Change List
- `infra/k3s/argocd/root-app.yaml` — App-of-Apps root Application
- `infra/k3s/argocd/apps/{metallb,traefik,cert-manager,external-secrets}.yaml` — child Application stubs
- `infra/k3s/ansible/playbooks/deploy-root-app.yml` — apply + wait Synced+Healthy
