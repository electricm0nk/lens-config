# Story 1.3: Join Worker Nodes

## Status: done

## Story

As an operator,
I want 5 worker nodes joined to the cluster via Ansible,
So that dedicated worker capacity is available for workload scheduling.

## Acceptance Criteria

- **Given** a 3-node HA control plane from Story 1.2
- **When** `ansible-playbook join-workers.yml` runs against the `k3s_workers` inventory group
- **Then** `kubectl get nodes` shows 5 additional nodes in `Ready` state with worker labels
- **And** all 8 total nodes report `Ready` simultaneously
- **And** no IP addresses are hardcoded in any playbook or inventory file (group names only)

## Tasks / Subtasks

- [x] Task 1: Create k3s-agent role
  - [x] `infra/k3s/ansible/roles/k3s-agent/tasks/main.yml` — download k3s binary as agent, install service with `--server` and token
  - [x] All tasks must have `name:` attributes
- [x] Task 2: Create join-workers.yml playbook
  - [x] `infra/k3s/ansible/playbooks/join-workers.yml` — targets `k3s_workers` group, applies `k3s-agent` role
  - [x] Token retrieved from first control-plane node at runtime (same pattern as Story 1.2)
  - [x] No hardcoded IPs

## Dev Notes

**Worker vs control-plane:** Worker nodes use `k3s agent` (not `k3s server`). The binary is the same but the subcommand differs. Pass `--server` URL and `--token` from the first control-plane node.

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Separate k3s-agent role (not k3s-server) — `ExecStart k3s agent`, agent config has only `server:` + `token:` (no CIDR, no cluster-init)
- kubelet port 10250 wait in role; kubectl node Ready wait in playbook post_tasks
- PR: electricm0nk/terminus.infra#26
### Change List
- `infra/k3s/ansible/roles/k3s-agent/tasks/main.yml` — worker role with k3s agent systemd service
- `infra/k3s/ansible/playbooks/join-workers.yml` — targets k3s_workers, runtime token slurp, 8-node assertion
