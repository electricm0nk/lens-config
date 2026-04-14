# Story 2.2: Create RBAC Baseline (ClusterRoles and ServiceAccounts)

## Status: done

## Story

As an operator,
I want ClusterRoles and ServiceAccounts created per the naming conventions,
So that components deployed later have the permissions they need from the start.

## Acceptance Criteria

- **Given** all 6 namespaces from Story 2.1
- **When** `ansible-playbook bootstrap-rbac.yml` applies RBAC manifests from `infra/k3s/manifests/rbac/`
- **Then** all expected ClusterRoles and ServiceAccounts are present in the cluster
- **And** all resources carry the 4 standard labels
- **And** no role grants `cluster-admin` to any non-operator ServiceAccount

## Tasks / Subtasks

- [x] Task 1: Create RBAC manifests
  - [x] `infra/k3s/manifests/rbac/clusterroles.yml` — ClusterRoles for ArgoCD, cert-manager, ESO (scoped, not cluster-admin)
  - [x] `infra/k3s/manifests/rbac/serviceaccounts.yml` — ServiceAccounts for each component in their respective namespaces
  - [x] All resources carry the 4 standard labels
- [x] Task 2: Create bootstrap-rbac.yml playbook
  - [x] `infra/k3s/ansible/playbooks/bootstrap-rbac.yml` — applies RBAC manifests, all tasks named
- [x] Task 3: Verify no cluster-admin grants
  - [x] Automated audit via kubectl + jq in playbook: confirms no SA has cluster-admin binding

## Dev Notes

**Standard labels on all resources:**
```yaml
labels:
  app.kubernetes.io/managed-by: ansible
  app.kubernetes.io/part-of: terminus-infra-k3s
  terminus.io/domain: terminus
  terminus.io/service: k3s
```

**Naming convention:** ServiceAccounts follow `{component}-sa` pattern (e.g., `argocd-sa`, `cert-manager-sa`, `external-secrets-sa`).

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Created operator-tier viewer ClusterRoles (not chart-managed) — avoids conflict with Helm RBAC
- cluster-admin audit uses `kubectl get clusterrolebinding -o json | jq` — catches implicit bindings via subjects[]
- PR: electricm0nk/terminus.infra#30
### Change List
- `infra/k3s/manifests/rbac/clusterroles.yml` — 3 scoped viewer ClusterRoles
- `infra/k3s/manifests/rbac/serviceaccounts.yml` — 3 operator SAs in respective namespaces
- `infra/k3s/ansible/playbooks/bootstrap-rbac.yml` — apply + verify + cluster-admin audit
