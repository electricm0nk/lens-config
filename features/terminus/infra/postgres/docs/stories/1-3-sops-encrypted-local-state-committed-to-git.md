# Story 1.3: SOPS-encrypted local state committed to git

Status: done

## Story

As an operator, I want OpenTofu state encrypted with my age key and committed to git, so that infrastructure state is version-controlled, never plaintext, and recoverable from git without a remote state backend.

## Acceptance Criteria

1. `terraform.tfstate.enc` is committed to git
2. Plaintext `terraform.tfstate` is never present in git at any point
3. `sops -d terraform.tfstate.enc > terraform.tfstate` succeeds before subsequent `tofu plan`
4. `.gitignore` includes `terraform.tfstate`

## Tasks / Subtasks

- [ ] Confirm `.gitignore` in `tofu/environments/dev/` excludes `terraform.tfstate` (AC: 2, 4)
  - [ ] Add `terraform.tfstate` and `terraform.tfstate.backup` entries if missing
- [ ] Document SOPS/age encryption workflow in dev environment README (AC: 3)
  - [ ] Pre-commit procedure: `sops -e terraform.tfstate > terraform.tfstate.enc && git add terraform.tfstate.enc`
  - [ ] Pre-apply procedure: `sops -d terraform.tfstate.enc > terraform.tfstate`
- [ ] Verify `.sops.yaml` age public key rules cover `terraform.tfstate.enc` files (AC: 1, 3)
- [ ] Perform initial SOPS encrypt cycle on current dev state (AC: 1)
  - [ ] Encrypt plaintext state: `sops -e terraform.tfstate > terraform.tfstate.enc`
  - [ ] Commit `terraform.tfstate.enc` to `terminus.infra`
  - [ ] Verify `git log` shows only `.enc` file, never plaintext
- [ ] Validate round-trip: decrypt → `tofu plan` shows no changes (AC: 3)

## Dev Notes

### Architecture Context

Per the architecture decision, OpenTofu state for this initiative uses local state + SOPS/age encryption rather than a remote backend. This was explicitly chosen for the homelab scale (see ADR-0004 in Story 1.4).

- State file: `tofu/environments/dev/terraform.tfstate`
- Encrypted state file: `tofu/environments/dev/terraform.tfstate.enc`
- SOPS config: `.sops.yaml` at `terminus.infra/` root (existing, references operator age key)
- State backend config: `backend "local" {}` in `versions.tf` or `main.tf`

### Responsibility Split

**This story does:**
- Ensures `.gitignore` excludes plaintext state
- Encrypts current state with SOPS/age and commits `.enc` file
- Documents the encrypt/decrypt workflow

**This story does NOT:**
- Create or modify the SOPS/age key infrastructure (existing tooling)
- Change backend type (stays `local`)
- Apply or change any infrastructure resources

### Directory Layout

```
terminus.infra/
  .sops.yaml                          # Existing — verify age key rules cover .enc files
  .gitignore                          # Root .gitignore
  tofu/
    environments/
      dev/
        .gitignore                    # MODIFY — add terraform.tfstate entries
        terraform.tfstate.enc         # CREATE — first SOPS-encrypted state commit
        README.md                     # CREATE/MODIFY — document encrypt/decrypt workflow
```

### Testing Standards

- `git diff HEAD -- terraform.tfstate` returns empty (plaintext not tracked)
- `sops -d terraform.tfstate.enc > terraform.tfstate && tofu plan` exits 0 with "No changes"
- `git log --all -- terraform.tfstate` shows no commits containing plaintext state
- `git show HEAD:tofu/environments/dev/terraform.tfstate.enc | head -5` shows SOPS MAC header

### References

- epics.md: Story 1.3 Acceptance Criteria
- architecture.md: State management — SOPS-local state, `terraform.tfstate.enc` encrypted with operator age key before git commit
- architecture.md: NFR-Additional — "no remote state backend for this capability"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/environments/dev/.gitignore` — MODIFY (add terraform.tfstate)
- `tofu/environments/dev/terraform.tfstate.enc` — CREATE (first encrypted state commit)
- `tofu/environments/dev/README.md` — CREATE or MODIFY (document SOPS workflow)
