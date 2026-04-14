# Story 2.3: Bootstrap ArgoCD via Ansible Helm Install

## Status: done

## Story

As an operator,
I want ArgoCD installed into the `argocd` namespace via Ansible running `helm install`,
So that the GitOps controller is running and able to manage subsequent component deployments.

## Acceptance Criteria

- **Given** the `argocd` namespace from Story 2.1 and RBAC baseline from Story 2.2
- **When** `ansible-playbook bootstrap-argocd.yml` runs `helm install argocd` with `infra/k3s/helm/argocd/values.yaml`
- **Then** all ArgoCD pods are `Running` in the `argocd` namespace
- **And** `helm ls -n argocd` shows the release in `deployed` state
- **And** `infra/k3s/helm/argocd/values.yaml` is the sole source of Helm configuration (no `--set` flags used in the playbook)
- **And** ArgoCD pods schedule only on worker nodes and carry no `NoSchedule` control-plane toleration

## Tasks / Subtasks

- [x] Task 1: Create ArgoCD Helm values
  - [x] `infra/k3s/helm/argocd/values.yaml` — configure server service type (ClusterIP); no nodeSelector/tolerations for control-plane
- [x] Task 2: Create bootstrap-argocd.yml playbook
  - [x] `infra/k3s/ansible/playbooks/bootstrap-argocd.yml` — runs `helm repo add`, `helm repo update`, `helm upgrade --install argocd argo/argo-cd -n argocd -f values.yaml`
  - [x] Uses `-f values.yaml` only — no `--set` flags
  - [x] All tasks named
- [x] Task 3: Wait for ArgoCD to be ready
  - [x] `kubectl wait --for=condition=available deployment -n argocd --all --timeout=300s`
- [x] Task 4: Verify no control-plane scheduling
  - [x] jq audit on pod specs confirms no CP toleration present

## Dev Notes

**Helm install command (no --set):**
```yaml
- name: Install ArgoCD via Helm
  ansible.builtin.command:
    cmd: helm install argocd argo/argo-cd --namespace argocd --values {{ playbook_dir }}/../helm/argocd/values.yaml --wait
```

**ArgoCD chart source:** `https://argoproj.github.io/argo-helm`

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Used `helm upgrade --install` for idempotency (re-runs won't fail on existing release)
- `--insecure` in values.yaml: ArgoCD serves plain HTTP; Traefik in Epic 3 handles TLS
- terminus.infra git repo registered in ArgoCD `configs.repositories` at install time
- PR: electricm0nk/terminus.infra#31
### Change List
- `infra/k3s/helm/argocd/values.yaml` — ClusterIP service, --insecure, terminus.infra repo config
- `infra/k3s/ansible/playbooks/bootstrap-argocd.yml` — helm install + wait + state assert + CP toleration audit
