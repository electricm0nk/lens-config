# Story 2.3: Secret Authoring Workflow (FR1, FR4, FR5)

Status: ready-for-dev

## Story

As the operator,
I want to create, version, and soft-delete secrets in Vault KV v2 via a documented SOPS-to-Vault authoring workflow,
so that all secret material is encrypted at rest in git and versioned in Vault with no plaintext exposure.

## Acceptance Criteria

1. **[SOPS encryption — no plaintext in git]** Given a plaintext secret value (e.g., `postgres app_role_password`), when I follow the secret authoring workflow (create `sops/postgres.sops.yaml`, encrypt with `sops -e`), then the resulting file in git contains no plaintext credentials — SOPS metadata header is present and all values are encrypted.

2. **[Decrypt with age key]** Given the encrypted `sops/postgres.sops.yaml`, when I run `sops -d sops/postgres.sops.yaml` with the correct age key, then the plaintext values are returned successfully.

3. **[KV v2 path follows hierarchy]** Given an encrypted `sops/postgres.sops.yaml` synced to Vault, when I run `vault kv get secret/terminus/infra/postgres/app_role_password`, then the current version is returned with the correct value and the path follows the `secret/terminus/{domain}/{service}/{key}` hierarchy exactly.

4. **[Version history retrievable (FR4)]** Given a Vault KV v2 secret that has been updated at least once, when I run `vault kv get -version=1 secret/terminus/infra/postgres/app_role_password`, then the previous version value is returned.

5. **[Soft-delete preserves version history (FR5)]** Given a Vault KV v2 secret, when I run `vault kv delete secret/terminus/infra/postgres/app_role_password`, then the secret cannot be read without a version flag (soft-deleted current version), but `vault kv get -version=1` still returns the historical value.

6. **[Secret authoring runbook complete]** Given `docs/runbooks/secret-authoring.md`, when I read it, then it covers the complete workflow: `sops -e` → `git commit` → sync to Vault KV v2 → verify read → optional soft-delete — with exact commands shown.

## Tasks / Subtasks

- [ ] **Task 1: Create `sops/postgres.sops.yaml` as the example secret file** (AC: 1, 2)
  - [ ] Create plaintext template:
    ```yaml
    # Keys: app_role_password
    app_role_password: "<POSTGRES_APPROLE_PASSWORD>"
    ```
  - [ ] Encrypt: `sops --encrypt --input-type yaml --output-type yaml /tmp/postgres-plain.yaml > sops/postgres.sops.yaml`
  - [ ] Alternatively use `sops sops/postgres.sops.yaml` (interactive) and add the key
  - [ ] Delete any plaintext temp file immediately
  - [ ] Verify no plaintext in committed file: `sops -d sops/postgres.sops.yaml | grep app_role_password` should require the age key

- [ ] **Task 2: Document the sync step (manual, pending Story 2.4 for automated)** (AC: 3)
  - [ ] For this story, document how to manually sync a SOPS value to Vault KV v2:
    ```bash
    vault kv put secret/terminus/infra/postgres/app_role_password \
      app_role_password="$(sops -d --extract '["app_role_password"]' sops/postgres.sops.yaml)"
    ```
  - [ ] This manual sync will be replaced by the OpenTofu automation in Story 2.4; document both the manual and automated paths in the runbook

- [ ] **Task 3: Demonstrate version history (FR4)** (AC: 4)
  - [ ] Run the sync twice with different values to create version 1 and version 2
  - [ ] Verify: `vault kv get -version=1 secret/terminus/infra/postgres/app_role_password` returns the v1 value
  - [ ] Document this command in the runbook

- [ ] **Task 4: Demonstrate soft-delete (FR5)** (AC: 5)
  - [ ] Run `vault kv delete secret/terminus/infra/postgres/app_role_password`
  - [ ] Verify `vault kv get` (current version) returns an error/empty
  - [ ] Verify `vault kv get -version=1` still returns the historical value
  - [ ] Undelete to restore state: `vault kv undelete -versions=2 secret/terminus/infra/postgres/app_role_password`

- [ ] **Task 5: Create `docs/runbooks/secret-authoring.md`** (AC: 6)
  - [ ] Sections:
    - `## Secret Authoring Workflow` — step-by-step: create plaintext → `sops -e` → delete plaintext → `git add` + `git commit` → sync to Vault
    - `## Path Hierarchy` — explain `secret/terminus/{domain}/{service}/{key}` with examples
    - `## Version History` — how to retrieve previous versions
    - `## Soft-Delete vs Destroy` — when to use `vault kv delete` (recoverable) vs `vault kv destroy` (permanent, Story 2.5)
    - `## Syncing to Vault` — manual sync command + note that Story 2.4 automates this in vault-config module
  - [ ] Cross-reference Story 2.5's destruction procedure for permanent deletion

## Dev Notes

### Architecture Constraints

- **Path hierarchy is a stable, breaking-change-sensitive contract:** `secret/terminus/{domain}/{service}/{key}` — NEVER rename paths silently. Path renames require coordinated migrations. For this story, the example uses `secret/terminus/infra/postgres/app_role_password` where domain=`infra`, service=`postgres`, key=`app_role_password`.
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy]

- **NFR1 — No plaintext on disk or in git:** The plaintext temp file must be deleted immediately after SOPS encryption. The committed file must contain only SOPS-encrypted content. Never commit a file that fails the `grep "sops:" sops/postgres.sops.yaml` check.
  [Source: docs/terminus/infra/secrets/prd.md — NFR1]

- **SOPS file format rules:** YAML format (not JSON), top-level keys match Vault path leaf names (`app_role_password`), plaintext key list in comment header.
  [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — SOPS File Format]

- **KV v2 path nuance:** In KV v2, the Vault provider and CLI use `secret/data/...` for actual data operations and `secret/metadata/...` for metadata operations. The `vault kv get` CLI command handles this transparently. In HCL policies and OpenTofu resources, use the `secret/data/terminus/...` and `secret/metadata/terminus/...` paths explicitly.
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy]

### Project Structure Notes

- New files: `sops/postgres.sops.yaml` (SOPS-encrypted), `docs/runbooks/secret-authoring.md`
- `sops/postgres.sops.yaml` is the reference example; additional service SOPS files follow the same pattern (`sops/{service}.sops.yaml`)
- The path `secret/terminus/infra/postgres/app_role_password` is the reference example for all hierarchy documentation

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.3]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — Vault Path Hierarchy, SOPS File Naming]
- [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — SOPS File Format]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
