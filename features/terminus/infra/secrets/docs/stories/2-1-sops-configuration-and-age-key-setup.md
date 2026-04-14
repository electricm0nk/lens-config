# Story 2.1: SOPS Configuration and Age Key Setup

Status: ready-for-dev

## Story

As the operator,
I want a `.sops.yaml` configuration file that specifies the age public key as the SOPS recipient for all `sops/*.sops.yaml` files,
so that any secret file created under `sops/` is automatically encrypted to the correct recipient without requiring per-file configuration.

## Acceptance Criteria

1. **[.sops.yaml creation rule]** Given the terminus.infra repository, when I inspect `sops/.sops.yaml`, then the file specifies the age public key (`age1...`) as the sole creation rule recipient, and the path rule matches `*.sops.yaml` files under the `sops/` directory.

2. **[sops/ directory tracked]** Given the `sops/` directory, when I run `git ls-files sops/`, then the directory appears in git tracking (either via `.gitkeep` or the `.sops.yaml` file itself).

3. **[Age key custody runbook exists]** Given `docs/runbooks/age-key-custody.md`, when I read it, then the runbook documents:
   - Where the age private key is stored (offline media, not network-accessible — NFR6)
   - How to verify the key is the correct recipient for the SOPS files
   - The custody transfer procedure if the key is compromised

4. **[Key loss recovery documented]** Given `docs/runbooks/age-key-custody.md`, when I read the "Key Loss" section, then it documents: all SOPS-encrypted files become unrecoverable if the age key is lost, and the only recovery path is restoring from a Vault Raft snapshot (if state was committed before key loss).

5. **[SOPS encrypts to correct recipient]** Given the `.sops.yaml` configuration, when I create a new file at `sops/test.sops.yaml` and run `sops sops/test.sops.yaml` to add a key, then the file is encrypted to the age public key specified in `.sops.yaml` — verified by the SOPS metadata header showing the age recipient.

## Tasks / Subtasks

- [ ] **Task 1: Create `sops/.sops.yaml`** (AC: 1)
  - [ ] Write `.sops.yaml` with path-based creation rule:
    ```yaml
    creation_rules:
      - path_regex: \.sops\.yaml$
        age: >-
          age1<OPERATOR_AGE_PUBLIC_KEY>
    ```
  - [ ] Replace `age1<OPERATOR_AGE_PUBLIC_KEY>` with the actual age public key for the terminus.infra deployment
  - [ ] Note: the age public key is NOT secret — it is safe to commit to git unencrypted

- [ ] **Task 2: Ensure `sops/` directory is tracked** (AC: 2)
  - [ ] The `sops/.sops.yaml` file itself satisfies the git-tracking requirement
  - [ ] Confirm `git ls-files sops/` shows the file after commit

- [ ] **Task 3: Create `docs/runbooks/age-key-custody.md`** (AC: 3, 4)
  - [ ] Write runbook covering:
    - `## Age Key Generation` — how to generate: `age-keygen -o key.txt`; key.txt contains both the public key (starts `# public key:`) and the private key (starts `AGE-SECRET-KEY-`)
    - `## Storage Requirements` — private key must be on offline media (USB/paper); must NOT be stored on the Vault VM or any network-accessible system (NFR6)
    - `## Custody Verification` — how to confirm the current key is the correct recipient: `sops -d sops/bootstrapcreds.sops.yaml` with `SOPS_AGE_KEY_FILE=key.txt` must succeed
    - `## Compromised Key Procedure` — if key is compromised: rotate immediately (see Story 2.6), re-encrypt all SOPS files with new key
    - `## Key Loss / Non-Recovery` — explicit statement: if age key is lost, ALL SOPS-encrypted files (including terraform.tfstate) are unrecoverable by SOPS; only recovery path is Vault Raft snapshot restoration (Story 5.2)
    - `## Bootstrap Credentials Reference` — note that `sops/bootstrapcreds.sops.yaml` contents are ephemeral and cleared after Day-0 bootstrap

- [ ] **Task 4: Add `sops/` to `.gitignore` exclusions review** (AC: 1)
  - [ ] Confirm `.gitignore` does NOT exclude `sops/.sops.yaml` or `sops/*.sops.yaml` — these ARE committed
  - [ ] The `.gitignore` excludes `*.age` (private key files) and `*.pem` but `.sops.yaml` and `*.sops.yaml` must be tracked
  - [ ] If `sops/` is accidentally excluded, fix `.gitignore`

- [ ] **Task 5: Verify SOPS round-trip** (AC: 5)
  - [ ] With `SOPS_AGE_KEY_FILE` set, run: `sops sops/test-verify.sops.yaml` (add key `test_value: hello`)
  - [ ] Inspect the encrypted file — confirm SOPS metadata header present, `age1...` recipient listed
  - [ ] `sops -d sops/test-verify.sops.yaml` — confirm decrypts successfully
  - [ ] Delete `sops/test-verify.sops.yaml` after verification (do not commit test files)

## Dev Notes

### Architecture Constraints

- **D1 — Age key delivery:** The age key is the trust anchor for the entire SOPS pipeline. Its custody is the highest-risk single point of failure. The private key must NEVER be committed to git, stored on the Vault VM, or stored on any network-accessible system. NFR6 is explicit: "age private key must never be stored on a network-accessible system unencrypted; custody location documented."
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D1]
  [Source: docs/terminus/infra/secrets/prd.md — NFR6]

- **SOPS file naming convention:** All SOPS files use `.sops.yaml` suffix — NOT `.enc.yaml`. The `.sops.yaml` suffix is mandatory per architecture naming rules. The SOPS config file is `sops/.sops.yaml` (inside the `sops/` directory, not at repo root).
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — SOPS File Naming]

- **Age public key is NOT secret:** The age public key (`age1...`) is safe to commit in `sops/.sops.yaml`. Only the private key (starting `AGE-SECRET-KEY-`) is sensitive.

- **`path_regex` in .sops.yaml:** The `path_regex` should match `\.sops\.yaml$` to apply the age key to all SOPS files in the repo. If you want to scope it to `sops/` directory only, use `sops/.*\.sops\.yaml$`. The architecture defines all SOPS files in the `sops/` directory — use the scoped pattern.

### Project Structure Notes

- New files created by this story:
  - `sops/.sops.yaml` — committed unencrypted (public key only)
  - `docs/runbooks/age-key-custody.md`
- The `docs/runbooks/` directory was created in Story 1.8. If Story 1.8 hasn't run yet, create the directory here.
- `sops/` directory is at the terminus.infra repository root per architecture.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 2.1]
- [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D1]
- [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — SOPS File Naming]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
