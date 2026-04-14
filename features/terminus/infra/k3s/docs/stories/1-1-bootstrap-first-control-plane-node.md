# Story 1.1: Bootstrap First Control-Plane Node

## Status: done

## Story

As an operator,
I want the first control-plane node initialized via Ansible with embedded etcd,
So that a k3s cluster foundation exists and kubeconfig is accessible from my workstation.

## Acceptance Criteria

- **Given** `infra/k3s/ansible/inventory/hosts.yml` defines a `k3s_control_plane` group using hostnames (no hardcoded IPs)
- **When** `ansible-playbook bootstrap.yml` runs against the first control-plane host
- **Then** k3s is installed with `--cluster-init`, embedded etcd is running, and kubeconfig is downloaded to `~/.kube/config` on the admin workstation
- **And** `kubectl get nodes` returns 1 node in `Ready` state with `control-plane` role
- **And** no k3s token or credential is present in any committed file (generated at runtime only)
- **And** all Ansible tasks in the playbook have a `name:` attribute

## Tasks / Subtasks

- [x] Task 1: Create inventory structure
  - [x] `infra/k3s/ansible/inventory/hosts.yml` with `k3s_control_plane` and `k3s_workers` groups (hostnames only, no IPs)
  - [x] `infra/k3s/ansible/group_vars/all.yml` with k3s version, install dir, kubeconfig path
- [x] Task 2: Create k3s-server role
  - [x] `infra/k3s/ansible/roles/k3s-server/tasks/main.yml` — download k3s binary, install service, start with `--cluster-init` flag for first node
  - [x] `infra/k3s/ansible/roles/k3s-server/tasks/main.yml` — kubeconfig download task
  - [x] All tasks must have `name:` attributes
- [x] Task 3: Create bootstrap.yml playbook
  - [x] `infra/k3s/ansible/playbooks/bootstrap.yml` — targets first host in `k3s_control_plane` group, applies `k3s-server` role
  - [x] No hardcoded IPs anywhere
- [x] Task 4: Verify no secrets committed
  - [x] Confirm k3s token generated at runtime (not committed)
  - [x] Verify `.gitignore` excludes any generated kubeconfig or token files

## Dev Notes

**k3s cluster-init flag:** The first control-plane node must use `--cluster-init` to initialize the embedded etcd cluster. Subsequent control-plane nodes join with `--server` pointing at the first node's URL.

**k3s install method:** Use the official k3s binary download (not k3sup). Pin the version in `group_vars/all.yml`.

**Kubeconfig retrieval:** After install, fetch `/etc/rancher/k3s/k3s.yaml` from the first control-plane node and save locally. Replace `127.0.0.1` with the actual node hostname.

**Token handling:** k3s generates a node token at `/var/lib/rancher/k3s/server/node-token`. Never commit this — read it via Ansible during the join playbooks (Story 1.2 and 1.3).

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- k3s-server role designed dual-mode: `k3s_init_node: true` uses `cluster-init`, `false` uses `server:` URL — avoids role duplication for Stories 1.2/1.3
- Added `disable: traefik` to k3s config to prevent conflict with Helm-managed Traefik (Epic 3)
- kubeconfig: fetched from `/etc/rancher/k3s/k3s.yaml`, 127.0.0.1 replaced with inventory hostname before saving locally
- PR: electricm0nk/terminus.infra#24
### Change List
- `infra/k3s/ansible/inventory/hosts.yml` — 3 CP nodes + 5 workers
- `infra/k3s/ansible/group_vars/all.yml` — k3s version, paths, CIDRs, MetalLB pool, ArgoCD config
- `infra/k3s/ansible/roles/k3s-server/tasks/main.yml` — dual-mode install role
- `infra/k3s/ansible/playbooks/bootstrap.yml` — first CP node bootstrap playbook
- `.gitignore` — k3s runtime artifact exclusions
