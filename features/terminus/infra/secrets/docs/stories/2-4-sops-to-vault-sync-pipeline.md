# Story 2.4: SOPS-to-Vault Sync Pipeline (FR2, FR3)

Status: ready-for-dev

## Story

As the operator,
I want an explicit, documented sync step that reads SOPS-encrypted files and writes their values into Vault KV v2 at the correct path,
so that the git source of truth and Vault runtime state are kept in sync via a repeatable, auditable operation.

## Acceptance Criteria

1. **[sops_file data source syncs to Vault]** Given a valid `sops/postgres.sops.yaml` encrypted with the age key, when the OpenTofu `vault-config/` module is applied (with `SOPS_AGE_KEY_FILE` set), then the secret value is written to Vault KV v2 at `secret/terminus/infra/postgres/app_role_password`.

2. **[No plaintext in state or on disk]** Given the sync operation, when I inspect `terraform.tfstate` (after SOPS encryption — Story 1.4), then the secret value is not readable as plaintext (it appears encrypted in state via SOPS). The OpenTofu resource uses the `sops_file` data source to decrypt transiently — no plaintext written to disk outside of SOPS-encrypted state.

3. **[Workload can read synced secret (FR3)]** Given an AppRole identity for a workload (Epic 3), when the workload authenticates and requests `vault kv get secret/terminus/infra/postgres/app_role_password`, then Vault returns the secret value and the audit log records the read (FR17).

4. **[Read latency within SLA (NFR11)]** Given a normal operating state, when a workload performs `vault kv get`, then the response is returned in < 2 seconds.

5. **[Sync is idempotent (NFR16)]** Given the vault-config module applied once, when I run `tofu plan` on vault-config, then the plan shows no changes (the `vault_kv_secret_v2` resource is idempotent).

## Tasks / Subtasks

- [ ] **Task 1: Add `data "sops_file" "postgres"` to `modules/vault-config/main.tf`** (AC: 1, 2)
  - [ ] Add:
    ```hcl
    data "sops_file" "postgres" {
      source_file = "${path.root}/sops/postgres.sops.yaml"
    }
    ```
  - [ ] Note: `carlpett/sops` provider must be available in `vault-config/` scope — it is declared at root level and passed via `providers` argument if needed

- [ ] **Task 2: Add `vault_kv_secret_v2.postgres` resource** (AC: 1, 5)
  - [ ] Add:
    ```hcl
    resource "vault_kv_secret_v2" "postgres" {
      mount               = "secret"
      name                = "terminus/infra/postgres/app_role_password"
      delete_all_versions = false
      data_json = jsonencode({
        app_role_password = data.sops_file.postgres.data["app_role_password"]
      })
    }
    ```
  - [ ] Resource name follows naming convention: `vault_kv_secret_v2.postgres`
  - [ ] `delete_all_versions = false` — preserve version history on re-apply

- [ ] **Task 3: Configure `carlpett/sops` provider in vault-config scope** (AC: 1)
  - [ ] Confirm the `sops` provider is declared in root `main.tf` `required_providers` and `provider "sops" {}` block
  - [ ] Pass to `module "vault-config"` via `providers = { ..., sops = sops }` if provider aliasing is required
  - [ ] The `sops` provider needs `SOPS_AGE_KEY_FILE` in the environment — no additional configuration required for the provider block itself beyond ensuring the env var is set

- [ ] **Task 4: Add sops and vault provider variables to `modules/vault-config/variables.tf`** (AC: 1)
  - [ ] Confirm `vault-config` accepts `vault_addr` and `vault_token` (or uses AppRole auth) for the Vault provider connection
  - [ ] Note: vault-config uses a non-root token — AppRole-derived token or a short-lived operator token. The Vault provider config for this module should NOT use the bootstrap root token (which was revoked in Story 1.6)

- [ ] **Task 5: Verify AC3 — workload read after sync** (AC: 3, 4)
  - [ ] After applying vault-config: `vault kv get secret/terminus/infra/postgres/app_role_password`
  - [ ] Confirm value returned matches the SOPS source value
  - [ ] Check `tail -1 /var/log/vault/audit.log | python3 -m json.tool` — confirm read operation logged
  - [ ] Measure response time (NFR11 — should be well under 2 seconds for local Vault)

## Dev Notes

### Architecture Constraints

- **carlpett/sops provider:** This is `carlpett/sops`, NOT `hashicorp/sops`. Source: `carlpett/sops`. Declared in root `versions.tf` `required_providers` (Story 1.1) and pinned in `.terraform.lock.hcl`.
  [Source: docs/terminus/infra/secrets/epics.md#Story 1.1 AC3]

- **SOPS state encryption (D4):** The decrypted secret value from `data.sops_file` will appear in `terraform.tfstate` (as OpenTofu stores data source results). This is why the SOPS-encryption-before-commit guard (Story 1.4) is critical. After applying, always run `make encrypt-state` before committing state.
  [Source: docs/terminus/infra/secrets/architecture.md#State & Storage — D4]

- **KV v2 resource path nuance:** In the `vault_kv_secret_v2` resource, the `name` field does NOT include the `secret/data/` mount prefix — it is relative to the mount. Set `mount = "secret"` and `name = "terminus/infra/postgres/app_role_password"`. The provider constructs the full API path as `secret/data/terminus/infra/postgres/app_role_password`.

- **vault-config authentication:** After Story 1.6 revokes the root token, `vault-config/` must authenticate via a non-root mechanism. For the MVP with a single operator machine, acceptable approaches: (a) use an AppRole for `opentofu` workload (bootstrapped via one-time manual step after vault-core), (b) use a long-lived operator token that is itself SOPS-encrypted in state. Document the chosen approach clearly. The architecture does not prescribe which mechanism is used for vault-config's own Vault provider token — only that it must NOT be the root token.

- **Idempotency:** `vault_kv_secret_v2` is idempotent when applied with the same data values. Re-running `tofu apply` with the same SOPS file will produce no Vault changes and `tofu plan` will show no changes (NFR16).

- **SOPS_AGE_KEY_FILE must be set at apply time:** The `sops_file` data source requires the age private key to be available in the environment during `tofu apply`. If `SOPS_AGE_KEY_FILE` is not set, the data source will fail with a key-not-found error. Document this in the Day-0 runbook (Story 1.8).

### Project Structure Notes

- This story modifies `modules/vault-config/main.tf` and `modules/vault-config/variables.tf`.
- Uses `sops/postgres.sops.yaml` created in Story 2.3.
- After this story, the Vault KV path `secret/terminus/infra/postgres/app_role_password` is populated and readable by authorized AppRole identities (Epic 3).

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.4]
- [Source: docs/terminus/infra/secrets/architecture.md#State & Storage — D4]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy, OpenTofu Resource Naming]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
