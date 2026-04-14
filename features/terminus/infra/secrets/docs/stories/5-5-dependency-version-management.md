# Story 5.5: Dependency Version Management — Pin, Comment, and Upgrade Procedure (FR37, FR38, NFR21, NFR22, NFR23)

Status: ready-for-dev

## Story

As the operator,
I want all tool and provider versions pinned with documented rationale and a clear upgrade procedure,
so that the system is reproducible, upgrade risks are managed, and no component drifts to an untested version automatically.

## Acceptance Criteria

1. **[Provider versions are pinned in .terraform.lock.hcl]** Given the OpenTofu project's `.terraform.lock.hcl`, when I read it, then all provider blocks use exact version pins (e.g., `version = "2.3.0"`) with no floating constraints (e.g., `version = ">= 2.3"` that could silently upgrade). All providers used in the project must appear in the lock file.

2. **[vault_version variable has pin comment]** Given `variables.tf` (in the root module or vault-core module), when I inspect the `vault_version` variable, then it contains a comment reading: `# Pinned per architecture baseline 2026-03-24; upgrade via tofu plan → review → tofu apply`

3. **[Version exception entries use documented ops-log.md pattern]** Given `docs/ops-log.md`, when I read it, then there exists at least one example version exception entry in this format:
   ```markdown
   ## Version Exception — <component>
   - **Component:** <e.g., hashicorp/vault provider>
   - **Pinned Version:** <e.g., 4.5.0>
   - **Current Latest:** <e.g., 4.6.0>
   - **Reason:** <e.g., v4.6.0 introduced breaking change in vault_policy resource — see GitHub issue #XXXX>
   - **Target Resolution:** <e.g., Upgrade after v4.6.1 is released with fix>
   - **Date:** <date>
   ```

4. **[Upgrade procedure comment in variables.tf cross-references NFR22]** Given the `vault_version` variable or its neighboring comment, when I read `variables.tf`, then the comment cross-references NFR22: `# NFR22: do not upgrade Vault without reviewing release notes and running tofu plan`

5. **[Architecture baseline versions are documented]** Given `docs/terminus/infra/secrets/architecture.md` or a version manifest file, when I read it, then the following component versions are listed as the architecture baseline:
   - Vault: 1.21.4
   - OpenTofu: 1.11.0
   - SOPS: latest stable (or pinned version)
   - age: latest stable (or pinned version)
   - Consul: 1.22.5
   - ansible-core: 2.19
   These are the versions established during the project and represent the known-working baseline.

6. **[Lock file is committed to version control (NFR21)]** Given the repository, when I run `git ls-files .terraform.lock.hcl` (or equivalent), then the lock file is tracked in git — confirming it is version-controlled alongside the OpenTofu configuration (NFR21: reproducible deployments require the lock file to be committed).

## Tasks / Subtasks

- [ ] **Task 1: Audit .terraform.lock.hcl for floating constraints** (AC: 1)
  - [ ] Read `.terraform.lock.hcl` (in the root module or each module directory)
  - [ ] Inspect each provider block: `version = "X.Y.Z"` must be an exact version pin
  - [ ] If any provider uses `>= X.Y.Z` or `~> X.Y` without an exact pin, this is a concern — but the lock file itself pins to an exact version per `tofu lock` mechanism. Verify the `versions.tf` constraints are not excessively loose (e.g., `>= 0.1` is too loose even if the lock file pins today's version)
  - [ ] Ensure `.terraform.lock.hcl` is committed: `git add .terraform.lock.hcl`

- [ ] **Task 2: Add pin comment to `vault_version` variable** (AC: 2, 4)
  - [ ] Locate `vault_version` in `variables.tf` (or the module file where it is defined)
  - [ ] Add (or update) the comment:
    ```hcl
    variable "vault_version" {
      description = "Vault version to install on the VM"
      type        = string
      default     = "1.21.4"
      # Pinned per architecture baseline 2026-03-24; upgrade via tofu plan → review → tofu apply
      # NFR22: do not upgrade Vault without reviewing release notes and running tofu plan
    }
    ```

- [ ] **Task 3: Verify all baseline versions are documented** (AC: 5)
  - [ ] Read `docs/terminus/infra/secrets/architecture.md` toolchain section
  - [ ] Confirm all 6 components are listed with their pinned versions
  - [ ] If any are missing or listed as "latest stable" without a pinned version, determine the actual installed version and update the architecture doc:
    - SOPS version: `sops --version`
    - age version: `age --version`
  - [ ] Add a `## Toolchain Version Baseline` section to architecture.md if not already present:
    ```markdown
    ## Toolchain Version Baseline (2026-03-24)

    | Component | Pinned Version | Notes |
    |---|---|---|
    | Vault | 1.21.4 | |
    | OpenTofu | 1.11.0 | |
    | SOPS | <version> | |
    | age | <version> | |
    | Consul | 1.22.5 | |
    | ansible-core | 2.19 | |
    | hashicorp/vault provider | <version from lock file> | |
    | bpg/proxmox provider | <version from lock file> | |
    | carlpett/sops provider | <version from lock file> | |
    | hashicorp/tls provider | <version from lock file> | |
    | hashicorp/null provider | <version from lock file> | |
    ```

- [ ] **Task 4: Add version exception entry pattern to ops-log.md** (AC: 3)
  - [ ] Open `docs/ops-log.md`
  - [ ] Add a `## Version Exception Template` section with the documented format (per AC3)
  - [ ] Add at least one real or simulated example entry — if no real version exceptions exist yet, create a demonstrative example:
    ```markdown
    ## Version Exception — Example (hashicorp/vault provider)
    - **Component:** hashicorp/vault provider
    - **Pinned Version:** 4.5.0 (example)
    - **Current Latest:** 4.5.0 (no exception active at baseline)
    - **Reason:** No exception — using current latest. Entry demonstrates format.
    - **Target Resolution:** N/A
    - **Date:** 2026-03-24
    ```

- [ ] **Task 5: Verify lock file is committed** (AC: 6)
  - [ ] Run: `git ls-files .terraform.lock.hcl` → should output the file path
  - [ ] If not tracked: `git add .terraform.lock.hcl && git commit -m "chore(tofu): commit terraform lock file for reproducibility (NFR21)"`
  - [ ] Confirm `.gitignore` does NOT exclude `.terraform.lock.hcl` — this file MUST be committed

- [ ] **Task 6: Review provider version constraints in versions.tf** (AC: 1)
  - [ ] Locate `versions.tf` (or `required_providers` block in root module `main.tf`)
  - [ ] Review each provider's `version` constraint:
    - Acceptable: `version = "= 4.5.0"` (exact pin) or `version = "~> 4.5.0"` (patch-level flexibility)
    - Risky: `version = ">= 4.0"` (major version float — could silently upgrade to breaking changes)
  - [ ] Change any overly-loose constraints to `~> X.Y.Z` or exact `= X.Y.Z`
  - [ ] Run: `tofu init -upgrade=false` to confirm lock file is consistent with constraints

## Dev Notes

### Architecture Constraints

- **NFR21 — Pinned versions, lock file committed:** "Reproducible deployments require `.terraform.lock.hcl` to be committed to version control." Many engineers incorrectly gitignore the lock file. For this project, it MUST be committed. This is enforced in the root module and all submodules that perform their own `tofu init`.

- **NFR22 — Upgrade procedure documented:** "OpenTofu and provider upgrades must go through: `tofu init -upgrade` → `tofu plan` → review → `tofu apply`." The comment in `variables.tf` and the architecture doc must reference this. Do NOT use `tofu init -upgrade` and `tofu apply` in the same step without a plan review.

- **NFR23 — Operator tool versions documented:** "Operator tool versions (SOPS, age, Ansible) documented." Task 3 produces this documentation. Undocumented tool versions make it impossible to diagnose version-specific bugs after the fact.

- **FR37 — Reproducible deployments:** The combination of pinned provider versions (lock file) + pinned Vault version (`vault_version` variable) + documented toolchain baseline ensures that any operator can reproduce the same infrastructure from the same repository.

- **FR38 — Upgrade path documented:** The `vault_version` comment and the ops-log.md version exception format together constitute the upgrade documentation. An operator knows: (1) what version is running, (2) how to upgrade (tofu plan → review → apply), and (3) how to record any exceptions.

- **`~> X.Y.Z` semantics:** The `~> 4.5.0` (pessimistic constraint) in OpenTofu allows `4.5.x` (patch updates) but not `4.6.x` (minor version). This is a reasonable balance — patch updates usually contain only bug fixes and security patches. Use `~> X.Y` (without patch) to allow minor version updates (less safe). Use `= X.Y.Z` for maximum reproducibility.

### Project Structure Notes

- Modifies `variables.tf` (updates vault_version comment)
- Modifies `docs/terminus/infra/secrets/architecture.md` (adds or updates toolchain version baseline)
- Modifies `docs/ops-log.md` (adds version exception template and example)
- Ensures `.terraform.lock.hcl` is committed (no new file — ensures existing file is tracked)
- Potentially modifies `versions.tf` (tightens provider version constraints if needed)

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 5.5]
- [Source: docs/terminus/infra/secrets/architecture.md#Additional Requirements — FR37/FR38]
- [Source: docs/terminus/infra/secrets/architecture.md#Non-Functional Requirements — NFR21/NFR22/NFR23]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
