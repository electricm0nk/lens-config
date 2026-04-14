# Story 3.4: k3s AppRole Identity and Policy — Least-Privilege infra/k3s/\* Reader (FR6, FR7)

Status: ready-for-dev

## Story

As the operator,
I want a k3s-specific AppRole identity with a policy scoped strictly to `secret/terminus/infra/k3s/*`,
so that the k3s cluster can read only its own secrets without accessing postgres credentials or AI agent secrets.

## Acceptance Criteria

1. **[Policy file header matches convention]** Given the HCL policy file exists at `vault-config/policies/k3s.hcl`, when I inspect it, then the first line reads exactly: `# Policy for k3s AppRole — read-only on secret/terminus/infra/k3s/*`

2. **[Policy grants read+list on infra/k3s/\* only (FR7)]** Given the k3s policy is written to Vault, when I read `vault policy read k3s`, then it includes `capabilities = ["read", "list"]` on `secret/data/terminus/infra/k3s/*` and `secret/metadata/terminus/infra/k3s/*` — and explicitly does NOT grant access to `secret/data/terminus/infra/postgres/*` or any other infra sub-path.

3. **[k3s can read its own secrets]** Given a k3s AppRole token, when I run `vault kv get secret/terminus/infra/k3s/<any_existing_key>`, then the operation succeeds (unless the secret doesn't yet exist, in which case "no value found" is acceptable — the important thing is HTTP 200 on a populated path or the absence of 403 on the k3s paths).

4. **[k3s cannot read postgres/* — isolation enforced]** Given a k3s AppRole token, when I attempt `vault kv get secret/terminus/infra/postgres/app_role_password`, then the response is HTTP 403 Forbidden.

5. **[k3s cannot read openclaw/* — cross-namespace denied]** Given a k3s AppRole token, when I attempt `vault kv get secret/terminus/agent/openclaw/<any_path>`, then the response is HTTP 403 Forbidden.

6. **[Both denied requests appear in audit log]** Given the two denied access attempts from AC4 and AC5, when I search the audit log with `grep "permission denied" /var/log/vault/audit.log | tail -5`, then entries for both failed operations are present (FR17/FR20 — all operations logged).

## Tasks / Subtasks

- [ ] **Task 1: Create `vault-config/policies/k3s.hcl`** (AC: 1, 2)
  - [ ] Create file `modules/vault-config/policies/k3s.hcl`:
    ```hcl
    # Policy for k3s AppRole — read-only on secret/terminus/infra/k3s/*

    path "secret/data/terminus/infra/k3s/*" {
      capabilities = ["read", "list"]
    }

    path "secret/metadata/terminus/infra/k3s/*" {
      capabilities = ["read", "list"]
    }

    path "sys/leases/renew" {
      capabilities = ["update"]
    }
    ```
  - [ ] Note: k3s policy intentionally omits `secret/data/terminus/infra/postgres/*` — this scoping is the key isolation objective

- [ ] **Task 2: Add `vault_policy.k3s` resource to vault-config module** (AC: 2)
  - [ ] In `modules/vault-config/main.tf` (or `policies.tf`):
    ```hcl
    resource "vault_policy" "k3s" {
      name   = "k3s"
      policy = file("${path.module}/policies/k3s.hcl")
    }
    ```

- [ ] **Task 3: Add `vault_approle_auth_backend_role.k3s` resource** (AC: 3)
  - [ ] In `modules/vault-config/main.tf` (or `roles.tf`):
    ```hcl
    resource "vault_approle_auth_backend_role" "k3s" {
      backend        = "approle"
      role_name      = "k3s"
      token_policies = [vault_policy.k3s.name]
      token_ttl      = 86400    # 24 hours (per Story 3.2)
      token_max_ttl  = 2592000  # 30 days (per Story 3.2)
      secret_id_num_uses = 0
    }
    ```

- [ ] **Task 4: Add `approle_role_id_k3s` output** (AC: 3)
  - [ ] In `modules/vault-config/outputs.tf`:
    ```hcl
    output "approle_role_id_k3s" {
      description = "Vault AppRole role_id for the k3s cluster identity"
      value       = vault_approle_auth_backend_role.k3s.role_id
      sensitive   = false
    }
    ```

- [ ] **Task 5: Test k3s can read k3s paths** (AC: 3)
  - [ ] Ensure at least one test secret exists at `secret/terminus/infra/k3s/kubeconfig_admin` (or similar) — seed it:
    `vault kv put secret/terminus/infra/k3s/kubeconfig_admin value="placeholder"`
  - [ ] Log in: `vault write auth/approle/login role_id=$(tofu output approle_role_id_k3s) secret_id=<generated>`
  - [ ] Verify read: `vault kv get secret/terminus/infra/k3s/kubeconfig_admin` returns data

- [ ] **Task 6: Test isolation — postgres/ and openclaw/ are 403** (AC: 4, 5)
  - [ ] With k3s token: `vault kv get secret/terminus/infra/postgres/app_role_password` → confirm 403
  - [ ] With k3s token: `vault kv get secret/terminus/agent/openclaw/api_key` → confirm 403
  - [ ] Check audit log: `grep "permission denied" /var/log/vault/audit.log | tail -5` shows both entries (AC: 6)

## Dev Notes

### Architecture Constraints

- **Scoping principle:** The k3s policy MUST scope to `infra/k3s/*` only — not `infra/*`. This is deliberate per the isolation model. A k3s cluster should not be able to read postgres credentials even if both are infrastructure secrets.

- **Secret hierarchy:** The canonical paths for this initiative follow the structure established in Story 2.3:
  - `secret/terminus/infra/postgres/` — database credentials (opentofu and operator access only)
  - `secret/terminus/infra/k3s/` — k3s cluster secrets (k3s-only)
  - `secret/terminus/agent/openclaw/` — AI agent credentials (openclaw-only)
  This story's policy enforces the middle slice.

- **KV v2 path semantics:** Same pattern as Story 3.3 — always include both `secret/data/...` and `secret/metadata/...` paths in KV v2 policies.

- **secret_id handling:** Same pattern as Story 3.3 — generate secret_id outside Tofu to avoid state exposure.

- **Isolation verification:** Story 3.7 (verify-isolation.yml) will formally automate the isolation checks from Tasks 5 and 6 of this story. During this story, manual verification is sufficient. The verify-isolation playbook is developed concurrently (3.3–3.5 and 3.7 are concurrent per sprint planning note).

### Project Structure Notes

- Creates `modules/vault-config/policies/k3s.hcl` (new file)
- Modifies `modules/vault-config/main.tf` (adds vault_policy.k3s and vault_approle_auth_backend_role.k3s)
- Modifies `modules/vault-config/outputs.tf` (adds approle_role_id_k3s output)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.4]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — FR6/FR7]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
