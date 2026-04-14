# Story 4.1: Prepare ESO Vault Bootstrap Secret via Ansible

## Status: done

## Story

As an operator,
I want the ESO ServiceAccount JWT registered with the Vault Kubernetes auth backend via Ansible,
So that the ESO `SecretStore` can authenticate to Vault once ESO is deployed by ArgoCD.

## Acceptance Criteria

- **Given** Vault running externally at `vault.trantor.internal` with Kubernetes auth backend enabled (pre-condition from `terminus-infra-secrets`)
- **When** `ansible-playbook bootstrap-eso-secret.yml` runs
- **Then** the ESO ServiceAccount JWT is registered with Vault's Kubernetes auth backend via `kubectl create secret` (not committed to source control)
- **And** the secret exists in the `external-secrets` namespace at runtime
- **And** no Vault token, JWT, or credential appears in any committed file
- **And** the playbook is idempotent (safe to re-run)

## Tasks / Subtasks

- [x] Task 1: Create bootstrap-eso-secret.yml playbook
  - [x] `infra/k3s/ansible/playbooks/bootstrap-eso-secret.yml`
  - [x] Reads ESO ServiceAccount JWT from cluster at runtime via `kubectl create token` (projected, not legacy secret)
  - [x] Registers JWT with Vault Kubernetes auth backend via `vault write auth/kubernetes/...`
  - [x] All tasks named; no credentials in any committed file
- [x] Task 2: Idempotency
  - [x] `vault write` overwrites existing config — safe re-runs
- [x] Task 3: Vault path config
  - [x] `vault_addr` from `group_vars/all.yml` (`vault_addr: https://vault.trantor.internal`)

## Dev Notes

**Vault Kubernetes auth registration:**
```yaml
- name: Register ESO ServiceAccount with Vault Kubernetes auth
  ansible.builtin.command:
    cmd: >
      vault write auth/kubernetes/role/external-secrets
        bound_service_account_names=external-secrets-sa
        bound_service_account_namespaces=external-secrets
        policies=external-secrets-policy
        ttl=1h
  environment:
    VAULT_ADDR: "{{ vault_addr }}"
    VAULT_TOKEN: "{{ vault_token }}"  # passed via --extra-vars, never committed
```

**No secrets in source control (NFR2):** Vault token passed at runtime via `--extra-vars` or environment variable. Never in `group_vars` or any committed file.

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Used `kubectl create token` (projected token) not legacy SA secret — k3s v1.32 no longer auto-creates token secrets
- Vault auth/kubernetes/config step registers k3s cluster CA + API server with Vault; required before role registration
- k3s_api_server uses `groups['k3s_control_plane'][0]` — no hardcoded IPs (NFR6)
- PR: electricm0nk/terminus.infra#37
### Change List
- `infra/k3s/ansible/playbooks/bootstrap-eso-secret.yml` — Vault Kubernetes auth config + ESO SA role registration
