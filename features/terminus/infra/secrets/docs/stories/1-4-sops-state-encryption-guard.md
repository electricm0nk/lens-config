# Story 1.4: SOPS State Encryption Guard [AR-S3 HIGH]

Status: ready-for-dev

## Story

As the operator,
I want a Makefile `encrypt-state` target and a git pre-commit hook that prevents committing unencrypted `terraform.tfstate`,
so that sensitive state values (AppRole IDs, root tokens, TLS private keys) can never be accidentally exposed in the repository.

## Acceptance Criteria

1. **[encrypt-state target works]** Given the repository with SOPS+age configured, when I run `make encrypt-state`, then `terraform.tfstate` is SOPS-encrypted in-place using the configured age recipient, the encrypted file contains the SOPS metadata header, no plaintext sensitive values are readable, and the command exits 0 on success.

2. **[encrypt-state fails safely]** Given the `make encrypt-state` command, when SOPS encryption fails (e.g., age key not found), then the command exits non-zero and the original state file is not corrupted or lost.

3. **[Pre-commit hook blocks unencrypted state]** Given the repository has a `.git/hooks/pre-commit` hook installed, when I run `git add terraform.tfstate` where the file is NOT SOPS-encrypted (no SOPS metadata header), then the commit is blocked with the message: `ERROR: terraform.tfstate appears unencrypted. Run 'make encrypt-state' first.`

4. **[Pre-commit hook allows encrypted state]** Given a correctly SOPS-encrypted `terraform.tfstate`, when I run `git add terraform.tfstate && git commit`, then the pre-commit hook passes and the commit proceeds.

5. **[install-hooks target documented]** Given the repository README, when I read the Development Setup section, then instructions for installing the pre-commit hook are documented including the `make install-hooks` command.

6. **[install-hooks target implements hook installation]** Given the Makefile, when I run `make install-hooks`, then the pre-commit hook is copied/symlinked to `.git/hooks/pre-commit` and made executable.

## Tasks / Subtasks

- [ ] **Task 1: Add `encrypt-state` to Makefile** (AC: 1, 2)
  - [ ] Implement `encrypt-state` target using atomic write pattern:
    ```makefile
    encrypt-state:
    	@echo "Encrypting terraform.tfstate with SOPS..."
    	@sops --encrypt terraform.tfstate > terraform.tfstate.enc.tmp \
    		&& mv terraform.tfstate.enc.tmp terraform.tfstate \
    		|| (rm -f terraform.tfstate.enc.tmp && echo "ERROR: SOPS encryption failed" && exit 1)
    	@echo "Done. terraform.tfstate is now SOPS-encrypted."
    ```
  - [ ] Confirm the temp file pattern (`terraform.tfstate.enc.tmp`) is covered by `.gitignore` (add if missing)
  - [ ] Add `encrypt-state` to `.PHONY` and to `make help` output with description: "SOPS-encrypt terraform.tfstate before commit"

- [ ] **Task 2: Create `.git/hooks/pre-commit` hook script** (AC: 3, 4)
  - [ ] Create `scripts/pre-commit` at repo root (the install target copies this to `.git/hooks/pre-commit`)
  - [ ] Hook logic: if `terraform.tfstate` is staged (`git diff --cached --name-only | grep terraform.tfstate`), check for SOPS metadata (`grep -q "sops:" terraform.tfstate`); if not found, echo error and `exit 1`
  - [ ] Make `scripts/pre-commit` executable (`chmod +x`)

- [ ] **Task 3: Add `install-hooks` target to Makefile** (AC: 5, 6)
  - [ ] Implement:
    ```makefile
    install-hooks:
    	@cp scripts/pre-commit .git/hooks/pre-commit
    	@chmod +x .git/hooks/pre-commit
    	@echo "Pre-commit hook installed."
    ```
  - [ ] Add `install-hooks` to `.PHONY` and `make help`

- [ ] **Task 4: Update README Development Setup section** (AC: 5)
  - [ ] Add a "Development Setup" section to `README.md` documenting:
    1. Clone the repo
    2. Run `make install-hooks` to install the pre-commit state encryption guard
    3. Set `SOPS_AGE_KEY_FILE` environment variable before any `tofu apply`
    4. Run `make encrypt-state` before every `git add terraform.tfstate`
  - [ ] Cross-reference `docs/runbooks/age-key-custody.md` for age key setup (Story 2.1)

- [ ] **Task 5: Test the guard end-to-end** (AC: 3, 4)
  - [ ] Run `make install-hooks`
  - [ ] Create a dummy unencrypted `terraform.tfstate` with `{"version": 4}` as content
  - [ ] `git add terraform.tfstate` and attempt commit — confirm blocked with correct error message
  - [ ] Run `make encrypt-state` (with `SOPS_AGE_KEY_FILE` set) — confirm exits 0 and file now has SOPS header
  - [ ] `git add terraform.tfstate && git commit -m "test"` — confirm hook passes

## Dev Notes

### Architecture Constraints

- **D4 — SOPS state encryption (architecture decision):** This is a HIGH-priority adversarial review item (AR-S3). The Terraform state file contains: AppRole IDs, Vault tokens (short-lived but present), TLS private keys. None of these must appear unencrypted in git history. The pre-commit hook is the safety net.
  [Source: docs/terminus/infra/secrets/architecture.md#State & Storage — D4]
  [Source: docs/terminus/infra/secrets/architecture.md#Process Patterns — State Encrypt-Before-Commit Rule]

- **SOPS metadata detection:** The hook checks for `sops:` key in the YAML. A SOPS-encrypted file always contains a `sops:` top-level key with metadata. An unencrypted JSON state file does NOT contain this key. This is a reliable, simple check.

- **Atomic write pattern:** The `encrypt-state` target MUST use a temp file + atomic move (`mv`) pattern. If SOPS encryption fails mid-write, the original state must remain intact. Never overwrite in place with a pipe directly to `terraform.tfstate` — that would truncate the file before writing if the process fails.

- **Hook is not committed:** The `.git/hooks/` directory is excluded from git tracking by design. The `scripts/pre-commit` wrapper IS committed to the repo so new clones get the hook script — they just need to run `make install-hooks` to activate it.

- **SOPS_AGE_KEY_FILE prerequisite:** The `encrypt-state` target requires `SOPS_AGE_KEY_FILE` to be set (or `SOPS_AGE_KEY` directly). If SOPS cannot find the age recipient key at encryption time, it exits non-zero. The Makefile should use `$(SOPS_AGE_KEY_FILE)` or rely on the env var being set — no magic fallback.

- **AR-S3 context:** The adversarial review flagged this as HIGH because prior drafts only described the discipline as prose in the architecture doc. This story makes it a machine-enforced guard. An implementer who skips this story leaves the project in an insecure state.
  [Source: docs/terminus/infra/secrets/adversarial-review-report.md — S3]

### Project Structure Notes

- New files created by this story:
  - `scripts/pre-commit` — hook source, committed to git
  - Updates to `Makefile` (encrypt-state, install-hooks targets)
  - Updates to `README.md` (Development Setup section)
- The `scripts/` directory is at the repository root of `terminus.infra/`.
- `*.enc.tmp` should be added to `.gitignore` to prevent accidental staging of temp encryption outputs.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.4]
- [Source: docs/terminus/infra/secrets/architecture.md#State & Storage — D4]
- [Source: docs/terminus/infra/secrets/architecture.md#Process Patterns — State Encrypt-Before-Commit Rule]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
