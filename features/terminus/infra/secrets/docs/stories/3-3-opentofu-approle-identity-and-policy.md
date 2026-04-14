# Story 3.3: OpenTofu AppRole Identity and Policy — Least-Privilege infra/\* Reader (FR6, FR7, FR16)

Status: ready-for-dev

## Story

As the operator,
I want an OpenTofu-specific AppRole identity with a least-privilege policy granting read access to only `secret/terminus/infra/*`,
so that automated OpenTofu pipelines can access infrastructure secrets without broader Vault permissions.

## Acceptance Criteria

1. **[Policy file header matches convention]** Given the HCL policy file exists at `vault-config/policies/opentofu.hcl`, when I inspect it, then the first line reads exactly: `# Policy for opentofu AppRole — read-only on secret/terminus/infra/*`

2. **[Policy grants read+list on infra/\* (FR7)]** Given the opentofu policy is written to Vault, when I read `vault policy read opentofu`, then it includes `capabilities = ["read", "list"]` on path `secret/data/terminus/infra/*` and on path `secret/metadata/terminus/infra/*`.

3. **[Policy grants lease renewal (FR16)]** Given the opentofu policy, when I read `vault policy read opentofu`, then it includes `capabilities = ["update"]` on path `sys/leases/renew` (allows token renewal).

4. **[AppRole login succeeds and can read infra paths]** Given the opentofu AppRole role_id and secret_id are provisioned, when I authenticate via `vault write auth/approle/login role_id=<opentofu_role_id> secret_id=<opentofu_secret_id>`, then authentication succeeds and the returned token can read `vault kv get secret/terminus/infra/postgres/app_role_password`.

5. **[Cross-namespace read denied — isolation check]** Given an opentofu token, when I attempt `vault kv get secret/terminus/agent/openclaw/<any_path>`, then the response is HTTP 403 Forbidden (policy does not grant access to agent/openclaw namespace).

6. **[Terraform output exports role_id (FR6)]** Given the vault-config module is applied, when I run `tofu output approle_role_id_opentofu`, then a non-empty role_id string is returned (the static role identifier for use in pipeline configuration).

## Tasks / Subtasks

- [ ] **Task 1: Create `vault-config/policies/opentofu.hcl`** (AC: 1, 2, 3)
  - [ ] Create file `modules/vault-config/policies/opentofu.hcl`:
    ```hcl
    # Policy for opentofu AppRole — read-only on secret/terminus/infra/*

    path "secret/data/terminus/infra/*" {
      capabilities = ["read", "list"]
    }

    path "secret/metadata/terminus/infra/*" {
      capabilities = ["read", "list"]
    }

    path "sys/leases/renew" {
      capabilities = ["update"]
    }
    ```
  - [ ] Note: KV v2 read requires both `secret/data/...` AND `secret/metadata/...` paths for full access

- [ ] **Task 2: Add `vault_policy.opentofu` resource to vault-config module** (AC: 2, 3)
  - [ ] In `modules/vault-config/main.tf` (or `policies.tf`):
    ```hcl
    resource "vault_policy" "opentofu" {
      name   = "opentofu"
      policy = file("${path.module}/policies/opentofu.hcl")
    }
    ```

- [ ] **Task 3: Add `vault_approle_auth_backend_role.opentofu` resource** (AC: 4, 6)
  - [ ] In `modules/vault-config/main.tf` (or `roles.tf`):
    ```hcl
    resource "vault_approle_auth_backend_role" "opentofu" {
      backend        = "approle"
      role_name      = "opentofu"
      token_policies = [vault_policy.opentofu.name]
      token_ttl      = 3600    # 1 hour (per Story 3.2)
      token_max_ttl  = 86400   # 24 hours (per Story 3.2)
      secret_id_num_uses = 0   # unlimited uses (rotate via secret-id practices)
    }
    ```
  - [ ] Create a `vault_approle_auth_backend_role_secret_id` or document that secret_id generation is a manual/deployment step (NOT stored in state)

- [ ] **Task 4: Add `approle_role_id_opentofu` output to vault-config module** (AC: 6)
  - [ ] In `modules/vault-config/outputs.tf`:
    ```hcl
    output "approle_role_id_opentofu" {
      description = "Vault AppRole role_id for the OpenTofu workload identity"
      value       = vault_approle_auth_backend_role.opentofu.role_id
      sensitive   = false  # role_id is not sensitive; secret_id is sensitive
    }
    ```

- [ ] **Task 5: Test policy — AppRole login and infra path read** (AC: 4)
  - [ ] Run: `vault write auth/approle/login role_id=$(tofu output approle_role_id_opentofu) secret_id=<generated>`
  - [ ] Export returned token: `export VAULT_TOKEN=<token>`
  - [ ] Verify: `vault kv get secret/terminus/infra/postgres/app_role_password` returns data
  - [ ] Confirm token policies: `vault token lookup | grep policies` shows `[default opentofu]`

- [ ] **Task 6: Test isolation — cross-namespace 403** (AC: 5)
  - [ ] With the opentofu token from Task 5:
  - [ ] Attempt: `vault kv get secret/terminus/agent/openclaw/any_key`
  - [ ] Confirm HTTP 403 and "permission denied" error
  - [ ] Optionally verify audit log records the denied request with `error="permission denied"`

## Dev Notes

### Architecture Constraints

- **KV v2 path semantics:** In KV v2, reading a secret requires policy access to BOTH `secret/data/<path>` (actual value) AND `secret/metadata/<path>` (metadata, version history). Granting only `secret/data/*` breaks `vault kv list` and version commands. Always include both in any KV v2 policy.

- **secret_id handling:** The secret_id for AppRole authentication is analogous to a password and must NEVER be stored in Terraform state. Recommended approach: generate it with `vault write -f auth/approle/role/opentofu/secret-id` outside of Tofu (operational step), store it in the SOPS bootstrap creds file (per Story 2.2), and reference it via `data "sops_file"` in pipeline configuration. The `vault_approle_auth_backend_role_secret_id` resource CAN be used but writes the secret_id to Terraform state — review this decision.

- **FR16 — OpenTofu Vault integration:** OpenTofu uses the carlpett/sops provider and/or vault provider for secret reads during `plan`/`apply` runs. The opentofu policy must allow reading the infra/* path where SOPS-synced secrets reside.

- **Policy file structure:** The `vault-config/policies/` directory holds all HCL policy files. They are loaded via `file()` at apply time. Keep policy files alongside the module for version control coherence.

### Project Structure Notes

- Creates `modules/vault-config/policies/opentofu.hcl` (new file)
- Modifies `modules/vault-config/main.tf` (adds vault_policy.opentofu and vault_approle_auth_backend_role.opentofu)
- Modifies or creates `modules/vault-config/outputs.tf` (adds approle_role_id_opentofu output)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 3.3]
- [Source: docs/terminus/infra/secrets/architecture.md#AppRole Vault Integration — FR6/FR7]
- [Source: docs/terminus/infra/secrets/architecture.md#Secret Management — FR16]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
