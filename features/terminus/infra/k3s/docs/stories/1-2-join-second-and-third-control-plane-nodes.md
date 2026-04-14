# Story 1.2: Join Second and Third Control-Plane Nodes

## Status: done

## Story

As an operator,
I want the second and third control-plane nodes joined to form a 3-member etcd quorum,
So that the control plane is highly available and tolerates a single node failure.

## Acceptance Criteria

- **Given** Story 1.1 is complete (first control-plane running, kubeconfig accessible)
- **When** `ansible-playbook join-control-plane.yml` runs against control-plane nodes 2 and 3
- **Then** `kubectl get nodes` shows 3 nodes with `control-plane` role in `Ready` state
- **And** `etcdctl member list` reports 3 healthy members
- **And** all host references in the playbook use inventory group names, not IP literals

## Tasks / Subtasks

- [x] Task 1: Extend k3s-server role for join mode
  - [x] Add join task to `infra/k3s/ansible/roles/k3s-server/tasks/main.yml` — uses `--server https://{{ first_control_plane_host }}:6443` and reads node token from first node
  - [x] Differentiate init vs join via `when:` condition on inventory position or group var flag
- [x] Task 2: Create join-control-plane.yml playbook
  - [x] `infra/k3s/ansible/playbooks/join-control-plane.yml` — targets remaining hosts in `k3s_control_plane` (nodes 2 and 3)
  - [x] All tasks named; no hardcoded IPs or hostnames outside inventory
- [x] Task 3: Token handling
  - [x] Retrieve token from first control-plane node at runtime using Ansible fact/slurp
  - [x] Never write token to any file in the repository

## Dev Notes

**etcd quorum:** After joining nodes 2 and 3, the embedded etcd cluster has 3 members and can tolerate 1 failure. Verify with `etcdctl member list` using the kubeconfig.

**Join URL:** The `--server` flag must point to the first control-plane node. Use the inventory hostname, resolved at runtime. Do not hardcode the IP.

**Token retrieval pattern:**
```yaml
- name: Read k3s node token from first control-plane node
  ansible.builtin.slurp:
    src: /var/lib/rancher/k3s/server/node-token
  register: k3s_token_raw
  delegate_to: "{{ groups['k3s_control_plane'][0] }}"
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- k3s-server role join mode was already implemented in Story 1.1 (dual-mode design) — Task 1 complete with zero additional code
- Token slurp uses `run_once: true` on first host then `set_fact` distributes to all joining hosts
- PR: electricm0nk/terminus.infra#25
### Change List
- `infra/k3s/ansible/playbooks/join-control-plane.yml` — targets CP nodes 2+3, runtime token slurp, HA quorum assertion
