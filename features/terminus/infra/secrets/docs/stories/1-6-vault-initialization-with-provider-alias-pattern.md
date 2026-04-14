# Story 1.6: Vault Initialization with Provider Alias Pattern [AR-S1 HIGH]

Status: ready-for-dev

## Story

As the operator,
I want the `vault-core/` module to initialize Vault via `vault operator init`, capture the unseal key and root token, and configure the Vault provider alias so that subsequent OpenTofu resources authenticate correctly — all within a single `tofu apply`,
so that the root token bootstrapping sequence is fully automated with no manual CLI steps required.

## Acceptance Criteria

1. **[Init via null_resource]** Given Vault is installed and running (Story 1.5 complete), when `vault-core/` module is applied, then a `null_resource` with a `remote-exec` provisioner executes `vault operator init -key-shares=1 -key-threshold=1` on the VM.

2. **[Unseal key and root token captured]** Given the init null_resource executes successfully, then the unseal key and root token are captured from the init command output and stored as `sensitive` OpenTofu local values within the `vault-core/` module.

3. **[Vault unsealed before provider resources]** Given the captured unseal key, then a second `null_resource` immediately executes `vault operator unseal <unseal_key>` — this resource has `depends_on` pointing to the init null_resource, ensuring unseal happens before any `vault_*` provider resources attempt to connect.

4. **[Provider alias pattern]** Given the root `main.tf`, when I inspect the provider declarations, then:
   - A `provider "vault"` block with `alias = "bootstrap"` is configured using the root token captured from the init step
   - This aliased provider is passed to `module "vault-core"` via `providers = { vault = vault.bootstrap }`
   - The default (un-aliased) `provider "vault"` block is NOT configured with the root token (it is used for post-bootstrap AppRole auth in later stories)

5. **[Root token revoked as final step]** Given all Vault resources within the `vault-core/` module are applied, then the LAST resource in `vault-core/` is a `null_resource` that executes `vault token revoke <root_token>` on the VM, satisfying NFR4 (root token must not persist after bootstrap).

6. **[Apply succeeds end-to-end]** Given the full root module, when I run `tofu apply`, then the command exits 0 with a running, unsealed, root-token-revoked Vault — verified by `vault status` returning `Sealed = false` and `Initialized = true`.

7. **[Code comment explains alias]** Given a reader of `vault-core/main.tf`, when they read the provider alias section, then an inline comment states:
   - `# root_token is captured from vault operator init stdout via null_resource.vault_init`
   - `# the vault.bootstrap alias exists ONLY during vault-core provisioning and is revoked as the final step`

## Tasks / Subtasks

- [ ] **Task 1: Implement `null_resource.vault_init` in `modules/vault-core/main.tf`** (AC: 1, 2)
  - [ ] Use `remote-exec` provisioner that:
    1. Runs `vault operator init -key-shares=1 -key-threshold=1 -format=json` and captures output to a temp file on the VM
    2. Output JSON is parsed to extract `unseal_keys_b64[0]` and `root_token`
    - Note: `remote-exec` output is captured to stdout in OpenTofu; use `local_sensitive_file` data source or `terraform_data` resource to pass values through OpenTofu state after capture

- [ ] **Task 2: Implement `null_resource.vault_unseal`** (AC: 3)
  - [ ] `depends_on = [null_resource.vault_init]`
  - [ ] `remote-exec`: run `vault operator unseal <unseal_key>` — unseal key passed as sensitive environment variable to the provisioner
  - [ ] Verify `vault status` returns `Sealed = false` after this step (can be added as a check)

- [ ] **Task 3: Configure `provider "vault" { alias = "bootstrap" }` in root `main.tf`** (AC: 4)
  - [ ] Declare the bootstrap provider alias with `address = var.vault_addr` and `token = <captured root token>`
  - [ ] The root token is an output of `module.vault-core` (sensitive) — but since it's needed for the provider config and providers are evaluated before module resources, use a `null_resource` ordering trick or pass via `depends_on` on provider configuration
  - [ ] **Implementation note:** This is the HIGH complexity S1 item from the adversarial review. The standard approach is to use a two-pass apply or to use `terraform_data` ephemeral resources. Document the chosen approach with a comment. A practical approach for single-node homelab: write the root token to a sensitive local_sensitive_file, use `external` data source or `file()` function with a `depends_on`. Alternatively, use a two-phase Makefile target (`tofu apply -target module.infra && tofu apply`) where phase 2 wires the provider alias.
  - [ ] Add inline comments as required by AC 7

- [ ] **Task 4: Implement `null_resource.vault_revoke_root_token` as LAST resource** (AC: 5)
  - [ ] `depends_on` on all other `vault_*` resources in `vault-core/`
  - [ ] `remote-exec`: `vault token revoke $VAULT_TOKEN` where `VAULT_TOKEN` is the root token
  - [ ] This is the FINAL resource — no Vault provider resources should `depends_on` this

- [ ] **Task 5: Update `modules/vault-core/outputs.tf`** (AC: 6)
  - [ ] Replace `vault_running` stub with `vault_initialized` (bool, value `true` — static signal after null_resource completion)
  - [ ] Root module uses `depends_on = [module.vault-core]` (which depends on `vault_initialized` completion) to gate `module.vault-config`

- [ ] **Task 6: Verify end-to-end** (AC: 6)
  - [ ] Run full `tofu apply`
  - [ ] Confirm: `vault status` → `Initialized = true`, `Sealed = false`
  - [ ] Confirm: `vault token lookup` with root token fails (already revoked)
  - [ ] Document unseal key capture instruction for operator: the unseal key appears in `tofu apply` stdout — operator must save it to offline custody per `docs/runbooks/age-key-custody.md`

## Dev Notes

### Architecture Constraints

- **D2 — Vault initialization strategy:** `null_resource` with `remote-exec` executes `vault operator init`. Init output (unseal keys + root token) emitted to `tofu apply` stdout for operator capture. Root token is passed into the Vault provider configuration for subsequent resource creation, then revoked as the final resource in `vault-core/`.
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2]

- **D3 — Unseal key strategy:** 1 key share, 1 threshold (`-key-shares=1 -key-threshold=1`). This is a documented deviation from NIST SP 800-57 dual-control (acknowledged in PRD risk register). The unseal key goes to offline custody per Story 2.1 runbook.
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D3]

- **NFR4 — Root token must not persist:** Root token revocation is the FINAL step. There must be no OpenTofu resource or provisioner that uses the root token after the revoke step. The architecture is explicit: "Root token is passed directly into the Vault provider configuration for subsequent resource creation, then revoked as the final resource in vault-core."
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2, NFR4]

- **AR-S1 HIGH — Provider alias complexity:** The Vault provider bootstrap alias pattern is the highest-complexity item flagged in the adversarial review. The two-pass apply approach (via Makefile) is acceptable for a homelab. Document the chosen approach clearly. The key constraint: OpenTofu providers are evaluated at plan time, but the root token is not available until `vault operator init` runs. Solutions: ephemeral resources (OpenTofu 1.11.0 supports this), two-pass apply, or `terraform_data` + `depends_on` gymnastics. Choose the simplest approach that works and document why.
  [Source: docs/terminus/infra/secrets/adversarial-review-report.md — S1]

- **`-format=json` on init:** Using `-format=json` makes parsing the root token deterministic and avoids fragile text parsing. The JSON output contains `root_token` and `unseal_keys_b64` fields.

- **Operator obligation:** The `tofu apply` stdout will contain the root token and unseal key. The operator MUST capture the unseal key before the terminal session closes. Add a prominent note in the Day-0 runbook (Story 1.8) and in inline comments.

- **Break-glass token lifecycle rule:** If the root token is ever re-used (break-glass), Story 5.3 governs the lifecycle. This story only covers Day-0 bootstrap revocation.
  [Source: docs/terminus/infra/secrets/architecture.md#Process Patterns — Break-Glass Token Lifecycle Rule]

### Project Structure Notes

- This story modifies `modules/vault-core/main.tf`, `modules/vault-core/outputs.tf`, and root `main.tf` (provider alias block).
- The `vault.bootstrap` alias provider is declared in root `main.tf` only. It does NOT appear in any module's `versions.tf` — provider requirements flow down from root.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.6]
- [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2, D3]
- [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
