# Story 1.4: Apply Control-Plane NoSchedule Taints

## Status: done

## Story

As an operator,
I want all 3 control-plane nodes tainted NoSchedule,
So that user workloads schedule exclusively on worker nodes.

## Acceptance Criteria

- **Given** 3 control-plane + 5 worker nodes from Stories 1.1–1.3
- **When** `ansible-playbook configure-nodes.yml` applies node taints
- **Then** each control-plane node carries taint `node-role.kubernetes.io/control-plane:NoSchedule`
- **And** a test `Pod` with no `tolerations` schedules only to worker nodes
- **And** no ArgoCD or ESO manifest in the repository includes a `NoSchedule` toleration for control-plane nodes

## Tasks / Subtasks

- [x] Task 1: Create configure-nodes.yml playbook
  - [x] `infra/k3s/ansible/playbooks/configure-nodes.yml` — applies `kubectl taint` via `command` module for each control-plane node
  - [x] Use inventory group `k3s_control_plane`; iterate members
  - [x] All tasks named
- [x] Task 2: Verify taint is applied
  - [x] Add verification task: `kubectl get node -o jsonpath='{.spec.taints}'` confirms taint present
- [x] Task 3: Audit ArgoCD and ESO manifests
  - [x] Grep all YAML files under `infra/k3s/` for `NoSchedule` toleration — confirm none present on ArgoCD/ESO resources

## Dev Notes

**Taint command:**
```yaml
- name: Apply NoSchedule taint to control-plane node
  ansible.builtin.command:
    cmd: kubectl taint node {{ item }} node-role.kubernetes.io/control-plane=:NoSchedule --overwrite
  loop: "{{ groups['k3s_control_plane'] }}"
```

**Idempotency:** Use `--overwrite` so re-running is safe when taint already exists.

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Playbook runs against `hosts: localhost` — all operations are kubectl calls; no SSH to cluster nodes needed
- Used `command` module with `--overwrite` flag for idempotency; `changed_when: false` since taint is verified separately
- Manifest audit excludes `ansible/` directory — only checks ArgoCD/Helm YAML under `infra/k3s/`
- PR: electricm0nk/terminus.infra#27
### Change List
- `infra/k3s/ansible/playbooks/configure-nodes.yml` — NoSchedule taint apply + verification + manifest audit
