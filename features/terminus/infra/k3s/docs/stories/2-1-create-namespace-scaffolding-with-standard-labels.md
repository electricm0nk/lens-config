# Story 2.1: Create Namespace Scaffolding with Standard Labels

## Status: done

## Story

As an operator,
I want all 6 cluster namespaces created via Ansible manifest apply,
So that component deployments have their target namespaces available before ArgoCD takes over.

## Acceptance Criteria

- **Given** a tainted 8-node cluster from Epic 1
- **When** `ansible-playbook bootstrap-namespaces.yml` runs
- **Then** all 6 namespaces exist: `argocd`, `cert-manager`, `external-secrets`, `metallb-system`, `terminus-infra`, `terminus-platform`
- **And** every namespace carries labels: `app.kubernetes.io/managed-by`, `app.kubernetes.io/part-of`, `terminus.io/domain`, `terminus.io/service`
- **And** `kubectl get ns` confirms all 6 in `Active` phase
- **And** all Ansible tasks in the playbook have a `name:` attribute

## Tasks / Subtasks

- [x] Task 1: Create namespace manifests
  - [x] `infra/k3s/manifests/namespaces/namespaces.yml` — defines all 6 namespaces with standard labels
  - [x] Labels: `app.kubernetes.io/managed-by: ansible`, `app.kubernetes.io/part-of: terminus-infra-k3s`, `terminus.io/domain: terminus`, `terminus.io/service: k3s`
- [x] Task 2: Create bootstrap-namespaces.yml playbook
  - [x] `infra/k3s/ansible/playbooks/bootstrap-namespaces.yml` — applies namespace manifest via `kubernetes.core.k8s`
  - [x] All tasks named
- [x] Task 3: Verify namespaces
  - [x] Add verification task confirming all 6 are in `Active` phase

## Dev Notes

**Namespace manifest pattern:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
  labels:
    app.kubernetes.io/managed-by: ansible
    app.kubernetes.io/part-of: terminus-infra-k3s
    terminus.io/domain: terminus
    terminus.io/service: k3s
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- `namespaces_manifest` path computed dynamically from `playbook_dir` — no hardcoded absolute paths
- Verification checks both Active phase and `managed-by=ansible` label per namespace
- PR: electricm0nk/terminus.infra#29
### Change List
- `infra/k3s/manifests/namespaces/namespaces.yml` — 6 Namespace objects with standard labels
- `infra/k3s/ansible/playbooks/bootstrap-namespaces.yml` — apply + verify playbook
