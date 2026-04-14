# Story 2.5: Secret Destruction Procedure (FR26)

Status: ready-for-dev

## Story

As the operator,
I want a documented, tested procedure for permanently destroying a secret — removing it from both Vault KV v2 and its SOPS source file in git,
so that secret destruction is a deliberate, auditable operation that eliminates all copies.

## Acceptance Criteria

1. **[All KV v2 versions permanently destroyed]** Given a secret at `secret/terminus/infra/postgres/app_role_password` in Vault, when I follow the destruction procedure, then `vault kv destroy` is executed with all version numbers (retrieved from metadata) permanently removing all versions — not a soft-delete.

2. **[KV v2 metadata removed]** Given the secret versions are destroyed, when I run `vault kv metadata delete secret/terminus/infra/postgres/app_role_password`, then the metadata record is also removed and subsequent `vault kv get` returns a 404.

3. **[SOPS source file deleted from git]** Given the destruction procedure, when I complete it, then `sops/postgres.sops.yaml` is deleted from git with a commit documenting the deletion — no git history squashing (the SOPS-encrypted history is safe to keep).

4. **[Final Vault read returns 404]** Given the complete destruction procedure is executed, when I run `vault kv get secret/terminus/infra/postgres/app_role_password`, then Vault returns a 404 — permanently destroyed, not soft-deleted.

5. **[Runbook documents procedure]** Given `docs/runbooks/secret-authoring.md` destruction section, when I read it, then it covers the exact destruction sequence with commands and notes the difference between `kv delete` (soft-delete, recoverable) and `kv destroy` (permanent, unrecoverable).

## Tasks / Subtasks

- [ ] **Task 1: Implement and test the vault-side destruction sequence** (AC: 1, 2, 4)
  - [ ] Get current version count: `vault kv metadata get -format=json secret/terminus/infra/postgres/app_role_password | jq .data.current_version`
  - [ ] Build version list: `$(seq 1 $current_version | tr '\n' ',' | sed 's/,$//')`
  - [ ] Destroy all versions: `vault kv destroy -versions=1,2,...,N secret/terminus/infra/postgres/app_role_password`
  - [ ] Delete metadata: `vault kv metadata delete secret/terminus/infra/postgres/app_role_password`
  - [ ] Verify: `vault kv get <path>` returns 404
  - [ ] Document the complete command sequence as a single copy-paste block in the runbook

- [ ] **Task 2: Remove SOPS source file from git** (AC: 3)
  - [ ] `git rm sops/postgres.sops.yaml`
  - [ ] `git commit -m "chore(sops): destroy postgres secret — all versions purged from Vault and git"`
  - [ ] Note in runbook: the SOPS-encrypted git history of the deleted file is safe to retain — an encrypted file in git history is only recoverable with the age key, and key custodians control that access

- [ ] **Task 3: Remove OpenTofu resource for the destroyed secret** (AC: 1, 5)
  - [ ] Remove `vault_kv_secret_v2.postgres` and `data "sops_file" "postgres"` from `modules/vault-config/main.tf`
  - [ ] Run `tofu plan` — state should show destroy of the removed resources
  - [ ] Run `tofu apply` — Vault resource is removed from state
  - [ ] Note: removing the OpenTofu resource and running `tofu apply` does NOT destroy KV v2 data by default (Vault resources are destroyed on apply, but KV secret delete behavior depends on `delete_all_versions` setting). The explicit `vault kv destroy` in Task 1 is the authoritative destruction step.

- [ ] **Task 4: Document destruction procedure in `docs/runbooks/secret-authoring.md`** (AC: 5)
  - [ ] Add `## Secret Destruction Procedure` section with:
    - Step 1: Determine all version numbers
    - Step 2: `vault kv destroy` all versions
    - Step 3: `vault kv metadata delete`
    - Step 4: Remove SOPS file from git (`git rm`)
    - Step 5: Remove OpenTofu resources from vault-config module
    - Step 6: `tofu apply` to reconcile state
    - Step 7: Verify 404 from Vault
    - Warning box: "This operation is irreversible. All version history is permanently deleted. Ensure this is the correct path before proceeding."
  - [ ] Add `## Soft-Delete vs Destroy` section explaining the difference

## Dev Notes

### Architecture Constraints

- **FR26 — Permanent destruction:** The requirement is permanent destruction — not soft-delete. `vault kv delete` is soft-delete (recoverable with `undelete`). `vault kv destroy` with all version numbers is permanent at the data level. `vault kv metadata delete` removes the metadata record entirely. Both commands are required.

- **Git history retention:** Deleting the SOPS-encrypted file from git HEAD does not erase git history. The encrypted historical versions remain in git history. This is acceptable — the SOPS-encrypted content is only recoverable with the age private key, which is in offline custody. Do NOT attempt to rewrite git history (no `git filter-branch` or force-push to remove the file from history) — that would be destructive to the git record.

- **OpenTofu state reconciliation:** After the `vault kv destroy` CLI operations, the Vault state managed by OpenTofu is inconsistent with reality (OpenTofu thinks the resource exists, Vault says it doesn't). Removing the resource from `vault-config/main.tf` and running `tofu apply` reconciles this cleanly.

- **Audit trail:** The destruction operations (both `vault kv destroy` and `vault kv metadata delete`) are recorded in the Vault audit log (FR17). This provides the system-side audit record of the destruction. The git commit message provides the operator-side record.

### Project Structure Notes

- This story modifies:
  - `modules/vault-config/main.tf` — removes postgres sops_file + kv resources
  - `docs/runbooks/secret-authoring.md` — adds destruction section
  - `sops/postgres.sops.yaml` — deleted from git (demonstrates the procedure)
- Note: removing `postgres.sops.yaml` via this story may be a demonstration that needs to be re-added for Epic 3 stories (which need postgres credentials). Coordinate with Epic 3 developer — either keep the file and only document the procedure, or use a separate test path that doesn't affect Epic 3 dependencies.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.5]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
