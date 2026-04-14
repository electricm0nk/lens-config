# Story 1.7: KV v2 Mount and AppRole Auth Backend

Status: ready-for-dev

## Story

As the operator,
I want the `vault-core/` module to enable the KV v2 secrets engine at `secret/` and enable the AppRole auth backend,
so that the mount points are ready before `vault-config/` provisions policies and identities.

## Acceptance Criteria

1. **[KV v2 secrets engine mounted]** Given Vault initialized and unsealed with root token available to provider alias, when `vault-core/` resources are applied, then `vault secrets list` includes a `secret/` path with engine type `kv` and version 2 — verified by `vault secrets get secret/` showing `options.version = 2`.

2. **[AppRole auth backend enabled]** Given the applied vault-core module, when I run `vault auth list`, then `approle/` is present with type `approle`.

3. **[Idempotent plan]** Given a clean apply of `vault-core/`, when I run `tofu plan` on vault-core, then the plan shows "No changes" (NFR16).

4. **[vault_initialized output]** Given `modules/vault-core/outputs.tf`, when I inspect it, then `vault_initialized` is declared as a boolean output that evaluates to `true` after all vault-core resources are applied — used by root module as `depends_on` signal for `module.vault-config`.

5. **[Provider alias used]** Given `modules/vault-core/main.tf`, when I inspect the resource declarations for `vault_mount` and `vault_auth_backend`, then they reference the `vault.bootstrap` provider alias (passed in via the `providers` argument from root module) — not the default (unauthenticated) provider.

## Tasks / Subtasks

- [ ] **Task 1: Add `vault_mount.secret` resource to `modules/vault-core/main.tf`** (AC: 1)
  - [ ] Add:
    ```hcl
    resource "vault_mount" "secret" {
      path    = "secret"
      type    = "kv"
      options = { version = "2" }
    }
    ```
  - [ ] Resource name follows architecture naming: `vault_mount.secret`
  - [ ] Ensure resource is declared BEFORE `null_resource.vault_revoke_root_token` (Story 1.6) in dependency order — use `depends_on = [null_resource.vault_unseal]` to gate after unseal

- [ ] **Task 2: Add `vault_auth_backend.approle` resource to `modules/vault-core/main.tf`** (AC: 2)
  - [ ] Add:
    ```hcl
    resource "vault_auth_backend" "approle" {
      type = "approle"
    }
    ```
  - [ ] Resource name: `vault_auth_backend.approle`
  - [ ] `depends_on = [null_resource.vault_unseal]` — must not run before unseal

- [ ] **Task 3: Update `null_resource.vault_revoke_root_token` depends_on** (AC: 5, Story 1.6 update)
  - [ ] The revoke-root-token null_resource must now `depends_on = [vault_mount.secret, vault_auth_backend.approle]`
  - [ ] This ensures root token is NOT revoked until BOTH engine and auth backend are provisioned
  - [ ] Order: unseal → vault_mount + vault_auth_backend → revoke root token

- [ ] **Task 4: Update `modules/vault-core/outputs.tf`** (AC: 4)
  - [ ] Declare `vault_initialized` output:
    ```hcl
    output "vault_initialized" {
      value       = true
      description = "Signal output — true when vault-core provisioning is complete (mounts + auth + root token revoked)"
      depends_on  = [null_resource.vault_revoke_root_token]
    }
    ```
  - [ ] This output is consumed by root `main.tf` via `module.vault-core.vault_initialized` in `depends_on` for `module.vault-config`

- [ ] **Task 5: Verify post-apply state** (AC: 1, 2, 3)
  - [ ] `vault secrets list` — confirm `secret/` present with kv/v2
  - [ ] `vault auth list` — confirm `approle/` present
  - [ ] `tofu plan` — confirm no changes
  - [ ] Record verification output in story completion notes

## Dev Notes

### Architecture Constraints

- **Implementation sequence (D2 → D1 prerequisite chain):** The architecture specifies the ordering: `infra/` (VM + TLS) → `vault-core/` (init, unseal, auth/mounts) → `vault-config/` (policies, AppRoles, secrets). This story completes the vault-core layer. The `vault_initialized = true` output is the formal signal that vault-core is complete.
  [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis — Implementation sequence]

- **vault_mount path = "secret":** The path is `secret` (no trailing slash in the HCL resource). Vault will serve it as `secret/`. The architecture specifies KV v2 at `secret/` — this is fixed, not configurable.
  [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints — Non-negotiable from PRD]

- **Provider alias handoff:** The `vault_mount` and `vault_auth_backend` resources use the `vault.bootstrap` alias provider (root token, short-lived). After `null_resource.vault_revoke_root_token` executes, the root token is gone. `module.vault-config/` will use a different authentication mechanism (AppRole-derived or another token). This is why vault-config is a separate module — it must authenticate independently.
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2]

- **Vault path hierarchy:** The `secret/` mount is the base for the `secret/terminus/{domain}/{service}/{key}` path convention. This mount must be KV v2 (not v1) — versioning, soft-delete, and metadata features are required by FRs 4 and 5.
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy]

- **NFR16 idempotency:** `vault_mount` and `vault_auth_backend` OpenTofu resources are naturally idempotent (provider manages lifecycle). `tofu plan` after clean apply should show no changes.

### Project Structure Notes

- This story modifies `modules/vault-core/main.tf` and `modules/vault-core/outputs.tf` only.
- No new files at repo root. No changes to root `main.tf` beyond the `depends_on` wiring (already established in Story 1.5/1.6).

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.7]
- [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy, AppRole Naming, Resource Naming]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
