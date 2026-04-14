# Story 1.1: Repository Skeleton and Provider Configuration

Status: ready-for-dev

## Story

As the operator,
I want a repository structure with all required OpenTofu provider configurations, version pins, and a .gitignore that prevents accidental secrets commit,
so that I have a reproducible, safe foundation for all subsequent OpenTofu work.

## Acceptance Criteria

1. **[Root file structure]** Given a clean `terminus.infra` repository clone, when I inspect the repository root, then these files exist: `main.tf`, `variables.tf`, `outputs.tf`, `Makefile`, `.gitignore`, `.terraform.lock.hcl`, and these directories exist: `modules/infra/`, `modules/vault-core/`, `modules/vault-config/`.

2. **[.gitignore coverage]** Given the repository root `.gitignore`, when I inspect its contents, then the following are excluded from tracking:
   - `.terraform/` directory
   - `*.tfstate.bak`
   - `*.tfstate` (unencrypted state files — `terraform.tfstate` is committed SOPS-encrypted)
   - `*.age` — plaintext age private-key files
   - `*.pem` — plaintext PEM key files outside explicitly committed paths

3. **[Provider version pins]** Given the committed `.terraform.lock.hcl` at the repository root, when I inspect it, then it pins all four required providers:
   - `hashicorp/proxmox` (latest compatible; version chosen at init time)
   - `hashicorp/vault` (version compatible with Vault 1.21.4)
   - `hashicorp/tls` (version pinned at init time)
   - `carlpett/sops` (version pinned at init time; source `carlpett/sops`)

4. **[Root variables declared]** Given `variables.tf` at the repository root, when I inspect its contents, then it declares exactly these input variables:
   - `proxmox_node` — the target Proxmox node name
   - `vault_version` — with default value `"1.21.4"`
   - `snapshot_host` — the off-VM snapshot target host (D5)
   - `sops_age_key_file` — path to the age private key file (D1); fed from `SOPS_AGE_KEY_FILE` env var

5. **[tofu init succeeds]** Given a fresh clone with `SOPS_AGE_KEY_FILE` set in the environment, when I run `tofu init`, then the command exits with code 0 and emits no provider version-drift warnings.

6. **[Module directory stubs]** Given the module directories, when I inspect `modules/infra/`, `modules/vault-core/`, and `modules/vault-config/`, then each contains at minimum `main.tf`, `variables.tf`, and `outputs.tf` (may be empty stubs with a `terraform {}` block to satisfy provider validation), and `modules/vault-config/policies/` directory exists.

7. **[Makefile help target]** Given the `Makefile`, when I run `make help`, then output lists at minimum these targets with one-line descriptions: `plan`, `apply`, `encrypt-state`, `verify`.

8. **[Provider isolation]** Given the root `main.tf` and all sub-module files, when I grep the provider usage across all `main.tf` files, then `hashicorp/proxmox` is referenced only within `modules/infra/` and `hashicorp/vault` is referenced only within `modules/vault-core/` and `modules/vault-config/` (no cross-layer provider bleed).

## Tasks / Subtasks

- [ ] **Task 1: Create root `versions.tf` with required_providers block** (AC: 3, 5)
  - [ ] Add `terraform { required_version = "~> 1.11.0" }` block (OpenTofu 1.11.0, NFR22)
  - [ ] Add `required_providers` block with `hashicorp/proxmox`, `hashicorp/vault`, `hashicorp/tls`, `carlpett/sops` — confirm exact minor version ranges are appropriate and will produce a deterministic `.terraform.lock.hcl`
  - [ ] Note in commit: which exact versions were locked and why (semver range chosen)

- [ ] **Task 2: Create root `variables.tf`** (AC: 4)
  - [ ] Declare `proxmox_node` (string, no default — required)
  - [ ] Declare `vault_version` (string, default `"1.21.4"`)
  - [ ] Declare `snapshot_host` (string, no default — required)
  - [ ] Declare `sops_age_key_file` (string, no default — set via `TF_VAR_sops_age_key_file` from `SOPS_AGE_KEY_FILE`)
  - [ ] All variables use lowercase snake_case names (naming rule from architecture)

- [ ] **Task 3: Create root `main.tf`** (AC: 1, 8)
  - [ ] Add provider configuration blocks for all four providers at root level (providers are _declared_ in root; sub-modules inherit via `providers` argument or root alias)
  - [ ] Add module call stubs: `module "infra" { source = "./modules/infra" }`, `module "vault-core"`, `module "vault-config"` (full wiring comes in later stories)
  - [ ] Ensure `hashicorp/proxmox` provider block is root-only; it is _not_ redeclared in `modules/vault-core/` or `modules/vault-config/`

- [ ] **Task 4: Create root `outputs.tf`** (AC: 1)
  - [ ] Add placeholder/stub outputs: `vault_addr`, `vault_ca_cert_pem`; mark with `# TODO: wire from module.infra outputs in Story 1.2/1.3`

- [ ] **Task 5: Create module stub directories and files** (AC: 6)
  - [ ] `modules/infra/main.tf` — empty body or `terraform {}` block; comment: `# Proxmox VM + networking + TLS cert — Story 1.2 / 1.3`
  - [ ] `modules/infra/variables.tf` — empty for now
  - [ ] `modules/infra/outputs.tf` — empty for now
  - [ ] `modules/vault-core/main.tf` — empty body; comment: `# Vault init/unseal/auth/mounts — Epic 2`
  - [ ] `modules/vault-core/variables.tf` — empty for now
  - [ ] `modules/vault-core/outputs.tf` — empty for now
  - [ ] `modules/vault-config/main.tf` — empty body; comment: `# Policies/AppRoles/audit — Epic 3`
  - [ ] `modules/vault-config/variables.tf` — empty for now
  - [ ] `modules/vault-config/outputs.tf` — empty for now
  - [ ] `mkdir -p modules/vault-config/policies` — directory for `{service}.hcl` files (Epic 3)

- [ ] **Task 6: Create `.gitignore`** (AC: 2)
  - [ ] Add `.terraform/` exclusion
  - [ ] Add `*.tfstate.bak` exclusion
  - [ ] Add `*.tfstate` exclusion (comment: SOPS-encrypted `terraform.tfstate` is committed explicitly)
  - [ ] Add `*.age` exclusion (plaintext age private keys must never reach git)
  - [ ] Add `*.pem` exclusion (plaintext PEM keys; CA public cert is `.crt`, not `.pem`)
  - [ ] Add `.terraform.lock.hcl` as a _tracked_ file (ensure it is NOT in .gitignore — it is intentionally committed)

- [ ] **Task 7: Create `Makefile`** (AC: 7)
  - [ ] Add `.PHONY` declaration for `help plan apply encrypt-state verify`
  - [ ] `help` target: print formatted list of all targets with descriptions (use `##` comment convention or explicit echo)
  - [ ] `plan` target: `tofu plan`
  - [ ] `apply` target: `tofu apply`
  - [ ] `encrypt-state` target: `sops --encrypt terraform.tfstate > terraform.tfstate.tmp && mv terraform.tfstate.tmp terraform.tfstate` (D4 discipline from architecture)
  - [ ] `verify` target: `ansible-playbook ansible/playbooks/verify-isolation.yml` (stub; playbook created in Epic 3/Story 5.x — target must exist with graceful no-op or helpful message if playbook is absent)

- [ ] **Task 8: Run `tofu init` and commit `.terraform.lock.hcl`** (AC: 3, 5)
  - [ ] Set `SOPS_AGE_KEY_FILE` env var (any valid age key file path — does not need to decrypt anything at init)
  - [ ] Run `tofu init` — confirm exit code 0
  - [ ] Inspect output for provider version-drift warnings — must be none
  - [ ] `git add .terraform.lock.hcl` — commit the generated lock file unencrypted (no secrets)
  - [ ] Confirm `.terraform/` directory is NOT tracked (verify .gitignore is working)

## Dev Notes

### Architecture Constraints

- **Toolchain version lock (NFR22):** `required_version = "~> 1.11.0"` in `versions.tf`. All operators must use OpenTofu 1.11.0. This is a hard pin — not a suggestion.
  [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints & Dependencies]

- **Module layer isolation (D2/architecture pattern):** Provider usage must not cross module boundaries. `hashicorp/proxmox` is instantiated in root `main.tf` and passed to `module "infra"` only. `hashicorp/vault` is instantiated in root `main.tf` but scoped to `module "vault-core"` and `module "vault-config"` via the `providers` argument. Verify with a grep pass before committing.
  [Source: docs/terminus/infra/secrets/architecture.md#Implementation Patterns & Consistency Rules]

- **Vault provider version compatibility:** The Vault provider version must be compatible with Vault OSS 1.21.4. Confirm via HashiCorp provider docs or changelog that the pinned provider version supports the Vault 1.21.4 API surface used in later stories (init, mount, auth, policy, KV v2, audit backend).
  [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints & Dependencies]

- **carlpett/sops provider:** This is `carlpett/sops`, NOT `hashicorp/sops`. The source must be `carlpett/sops` in `required_providers`. This provider is used in `vault-config/` for SOPS-decrypted secret injection (Epic 3). It is declared at root level here so the lock file includes it from Day 0.
  [Source: docs/terminus/infra/secrets/epics.md#Story 1.1]

- **State encryption discipline (D4):** `terraform.tfstate` is committed SOPS-encrypted. The `.gitignore` excludes `*.tfstate` but the Makefile `encrypt-state` target is the prescribed path. The state file is added to git _after_ running `make encrypt-state`. The `.terraform.lock.hcl` is NOT encrypted and is committed plaintext — it contains no secrets.
  [Source: docs/terminus/infra/secrets/architecture.md#State & Storage, Process Patterns]

- **SOPS_AGE_KEY_FILE delivery (D1):** The `sops_age_key_file` variable is the operator's mechanism for providing the age private key path to `tofu apply`. It should be fed via `TF_VAR_sops_age_key_file` (which echoes `SOPS_AGE_KEY_FILE`). Story 1.1 only declares the variable — the actual SOPS datasource calls are in Epic 3 (`vault-config/`). At this story, `tofu init` must succeed with any valid age key environment — key does not need to decrypt real secrets yet.
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery (D1)]

- **Makefile `verify` target stub:** The `ansible-playbook verify-isolation.yml` invocation is the full form (D8), but the playbook is created in Story 5.x. For Story 1.1, the `verify` target should either print a `not yet implemented` message or invoke the playbook with a `--check` flag that gracefully handles file-not-found. Do NOT omit the target — it must be in `make help` output per AC 7.
  [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure (D8)]

### Project Structure Notes

- The repository root is `terminus.infra/` (target repo: `electricm0nk/terminus.infra`). All file paths in this story are relative to that repo root.
- Module directories are under `modules/` — not at the repository root. This is a definitive architecture decision (Layered OpenTofu Modules — Selected Approach).
  [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]
- `modules/vault-config/policies/` must be created now (empty) to establish the directory contract. Policy `.hcl` files are added in Epic 3.
- No `sops/` directory is created in this story — that is part of Epic 3 (SOPS pipeline setup).
- No `ansible/` directory is created in this story — that is Epic 4/5 scope.

### OpenTofu Variable Naming Rules

All variable names are lowercase snake_case per architecture naming convention. The four root-level variables map directly to environment injection:

| Variable | Env var source | Default |
|---|---|---|
| `proxmox_node` | `TF_VAR_proxmox_node` | none (required) |
| `vault_version` | `TF_VAR_vault_version` | `"1.21.4"` |
| `snapshot_host` | `TF_VAR_snapshot_host` | none (required) |
| `sops_age_key_file` | `TF_VAR_sops_age_key_file` | none (required) |

[Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — OpenTofu Variable Naming]

### Commit Message Convention

```
feat(infra): repository skeleton — provider config, versions, and module stubs
```

[Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — Git Commit Messages]

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.1] — Acceptance Criteria (BDD scenarios)
- [Source: docs/terminus/infra/secrets/architecture.md#Starter Approach] — Layered OpenTofu Modules decision
- [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery (D1)] — SOPS_AGE_KEY_FILE age key delivery
- [Source: docs/terminus/infra/secrets/architecture.md#State & Storage (D4)] — State encryption before commit
- [Source: docs/terminus/infra/secrets/architecture.md#Operational Infrastructure (D8)] — Isolation verification suite (make verify stub)
- [Source: docs/terminus/infra/secrets/architecture.md#Implementation Patterns — Naming Patterns] — Variable naming, resource naming, file naming rules
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure] — Canonical terminus.infra directory layout

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
