# Story 2.2: Bootstrap Credentials Management

Status: ready-for-dev

## Story

As the operator,
I want a `sops/bootstrapcreds.sops.yaml` file that holds the bootstrap-phase credentials needed before Vault exists,
so that the Day-0 bootstrap process has a documented, encrypted location for ephemeral credentials that must not persist beyond their use.

## Acceptance Criteria

1. **[bootstrapcreds.sops.yaml encrypted correctly]** Given SOPS configured with age recipient (Story 2.1), when I run `sops -d sops/bootstrapcreds.sops.yaml` with the age private key, then decryption succeeds and the file contains documented comment headers describing what each key is for.

2. **[Commit comment documents key names]** Given the SOPS-encrypted `sops/bootstrapcreds.sops.yaml` committed to git, when I read the plaintext comment at the top of the encrypted file (visible without decryption), then it documents the key names: `# Keys: <list of keys stored in this file>`.

3. **[Commit message follows convention]** Given the git commit that adds `bootstrapcreds.sops.yaml`, when I inspect the commit message, then it follows the pattern: `chore(sops): add bootstrap credentials template`.

4. **[Age-key-custody runbook references bootstrapcreds]** Given `docs/runbooks/age-key-custody.md`, when I read the section on bootstrap credentials, then it is documented that `bootstrapcreds.sops.yaml` contents are ephemeral and should be cleared or rotated after a successful Day-0 bootstrap.

5. **[File is tracked in git]** Given `sops/bootstrapcreds.sops.yaml`, when I run `git show HEAD:sops/bootstrapcreds.sops.yaml`, then the SOPS-encrypted file content is returned (not a missing file error).

## Tasks / Subtasks

- [ ] **Task 1: Create `sops/bootstrapcreds.sops.yaml`** (AC: 1, 2)
  - [ ] Create the plaintext template (`/tmp/bootstrapcreds-plain.yaml`):
    ```yaml
    # Keys: proxmox_api_token, proxmox_api_url
    # EPHEMERAL: Clear these values after successful Day-0 bootstrap
    proxmox_api_token: "<SET_BEFORE_BOOTSTRAP>"
    proxmox_api_url: "<SET_BEFORE_BOOTSTRAP>"
    ```
  - [ ] Encrypt with SOPS: `sops --encrypt /tmp/bootstrapcreds-plain.yaml > sops/bootstrapcreds.sops.yaml`
  - [ ] Delete the plaintext temp file immediately after encryption
  - [ ] Verify the SOPS header in the resulting file: `head -5 sops/bootstrapcreds.sops.yaml` should show SOPS metadata but no plaintext values

- [ ] **Task 2: Add `# Keys:` plaintext comment**  (AC: 2)
  - [ ] SOPS preserves top-level YAML comments in encrypted output — the `# Keys:` comment line should appear in the encrypted file's first few lines
  - [ ] Verify: `head -3 sops/bootstrapcreds.sops.yaml | grep "# Keys:"` returns a match

- [ ] **Task 3: Verify round-trip decrypt** (AC: 1)
  - [ ] `SOPS_AGE_KEY_FILE=key.txt sops -d sops/bootstrapcreds.sops.yaml`
  - [ ] Confirm decrypted output shows the key names and placeholder values
  - [ ] Confirm no errors

- [ ] **Task 4: Commit with correct message** (AC: 3, 5)
  - [ ] `git add sops/bootstrapcreds.sops.yaml`
  - [ ] `git commit -m "chore(sops): add bootstrap credentials template"`
  - [ ] Confirm with `git show HEAD:sops/bootstrapcreds.sops.yaml` returns content

- [ ] **Task 5: Update `age-key-custody.md`** (AC: 4)
  - [ ] Add section to `docs/runbooks/age-key-custody.md`:
    > **Bootstrap Credentials:** `sops/bootstrapcreds.sops.yaml` holds ephemeral Day-0 credentials (Proxmox API token, etc.). After a successful `tofu apply` on Day-0, these credentials should be rotated or cleared — they are only needed during initial VM provisioning. Rotate by: `sops sops/bootstrapcreds.sops.yaml` → set values to empty or new rotated values → save → `git add ... && git commit`.

## Dev Notes

### Architecture Constraints

- **SOPS file naming convention:** `bootstrapcreds.sops.yaml` — the `.sops.yaml` suffix is mandatory. The file lives in `sops/` directory.
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — SOPS File Naming]

- **SOPS file format:** All SOPS files are YAML (not JSON). Top-level keys match Vault path leaf names where applicable. For bootstrap credentials that don't map to Vault paths, use descriptive names. Plaintext structure documented in comment at top of encrypted file.
  [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — SOPS File Format]

- **Ephemeral credential handling:** The bootstrapcreds file holds credentials needed BEFORE Vault exists (e.g., Proxmox API credentials for VM provisioning via OpenTofu). Once Vault is operational and provisioned, these credentials should be managed through Vault KV v2 like all other credentials, and the SOPS file cleared. This satisfies the principle of minimal long-term credentials.

- **NFR1 — Never plaintext on disk:** The encrypted file is committed. The plaintext temp file must be deleted immediately after SOPS encryption. Never `git add` the plaintext version.
  [Source: docs/terminus/infra/secrets/prd.md — NFR1]

- **Dependency on Story 2.1:** `sops/.sops.yaml` must exist (Story 2.1) before SOPS can encrypt with the correct age recipient. Story 2.2 depends on Story 2.1 being complete.

### Project Structure Notes

- New file: `sops/bootstrapcreds.sops.yaml` — SOPS-encrypted, committed
- Modifies: `docs/runbooks/age-key-custody.md` (bootstrap credentials section)
- No changes to OpenTofu files — this is a SOPS/documentation story

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.2]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — SOPS File Naming]
- [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — SOPS File Format]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
