# Story 1.8: Root Module Wiring and Day-0 Bootstrap Runbook

Status: ready-for-dev

## Story

As the operator,
I want the root `main.tf` to wire all three modules in correct dependency order with explicit `depends_on`, expose the required public contract outputs, and have a complete Day-0 runbook,
so that `tofu apply` from zero executes the full provisioning sequence, and a returning operator can execute Day-0 from documentation alone.

## Acceptance Criteria

1. **[Module dependency chain correct]** Given the root `main.tf`, when I inspect the module declarations, then `module "vault-core"` has `depends_on = [module.infra]` and `module "vault-config"` has `depends_on = [module.vault-core]`.

2. **[Root outputs expose public contract]** Given root `outputs.tf`, when I run `tofu output`, then the following are present:
   - `vault_addr` — `https://{vm_ip}:8200` (constructed from `module.infra.vm_ip`)
   - `vault_ca_cert_pem` — PEM string, not marked sensitive (public trust anchor)
   - `approle_role_id_{service}` outputs for each registered workload (wired from `module.vault-config` in Epic 3)
   - All sensitive outputs (private keys, tokens) are marked `sensitive = true`

3. **[tofu plan shows no changes after clean apply]** Given a fully applied root module, when I run `tofu plan`, then the output shows "No changes. Your infrastructure matches the configuration." (NFR16).

4. **[Day-0 runbook exists and is complete]** Given `docs/runbooks/day0-bootstrap.md`, when I read it with no prior context, then the runbook covers:
   - Prerequisites section: age key location, Proxmox credentials, OpenTofu installation (1.11.0)
   - Environment variable setup: `SOPS_AGE_KEY_FILE`, `TF_VAR_proxmox_*`, `TF_VAR_vault_version`, `TF_VAR_vm_ip`
   - Full command sequence: `make install-hooks`, `tofu init`, `tofu apply`
   - Operator obligation: "CAPTURE UNSEAL KEY FROM STDOUT — this is your only copy"
   - Post-apply verification steps: `vault status`, `curl -k https://{vault_addr}/v1/sys/health`, `tofu plan` (confirm no changes)
   - Post-apply: instruction to run `ansible-playbook verify-isolation.yml` after vault-config is applied (Epic 3)

5. **[Runbook documents vault_ca_cert_pem distribution]** Given `docs/runbooks/day0-bootstrap.md`, when I read the post-apply steps, then there is a step instructing the operator to: run `tofu output -raw vault_ca_cert_pem > ca.crt` and commit `ca.crt` to the repository so consumers can trust the Vault TLS certificate.

6. **[Full apply exits 0]** Given a clean environment with all env vars set, when I run `tofu apply`, then the apply completes with exit code 0, Vault is operational (Story 1.6 verified), and `tofu plan` subsequently shows no changes.

## Tasks / Subtasks

- [ ] **Task 1: Finalize root `main.tf` module wiring** (AC: 1)
  - [ ] Ensure `module "vault-core"` block has `depends_on = [module.infra]` (may already be set from Story 1.5)
  - [ ] Add `module "vault-config"` stub block with `depends_on = [module.vault-core]` (vault-config will be populated in Epic 3 stories)
  - [ ] Verify `module "vault-config"` receives `vault_initialized = module.vault-core.vault_initialized` as a variable if needed for ordering
  - [ ] Pass `providers` to each module as appropriate (vault.bootstrap to vault-core; regular vault provider to vault-config)

- [ ] **Task 2: Finalize root `outputs.tf`** (AC: 2)
  - [ ] Wire `vault_addr = "https://${module.infra.vm_ip}:8200"` (replacing stub from Story 1.1)
  - [ ] Wire `vault_ca_cert_pem = module.infra.ca_cert_pem`
  - [ ] Add stub outputs for `approle_role_id_opentofu`, `approle_role_id_k3s`, `approle_role_id_openclaw` with `# TODO: wire from module.vault-config in Story 3.3–3.5` comments
  - [ ] Add `description` field to every output (architecture contract: no silent outputs)

- [ ] **Task 3: Create `docs/runbooks/day0-bootstrap.md`** (AC: 4, 5)
  - [ ] Create `docs/runbooks/` directory
  - [ ] Write runbook with sections:
    - `## Prerequisites`
    - `## Environment Setup`
    - `## Bootstrap Execution`
    - `## Operator Obligation: Capture Unseal Key`
    - `## Post-Apply Verification`
    - `## CA Cert Distribution`
    - `## What to Verify After Bootstrap`
  - [ ] Reference `make install-hooks` (Story 1.4), `SOPS_AGE_KEY_FILE` (D1), vault status commands (Story 1.6), `verify-isolation.yml` (Story 3.7)
  - [ ] Include "What can go wrong" section: SOPS age key unavailable, Proxmox API unreachable, Vault init output not captured (Story 5.1 prerequisite)

- [ ] **Task 4: Run full end-to-end apply** (AC: 3, 6)
  - [ ] Fresh clone + all env vars set: `tofu apply`
  - [ ] Confirm exit code 0
  - [ ] Run `tofu plan` — confirm no changes
  - [ ] Run `vault status`, `vault secrets list`, `vault auth list`
  - [ ] Record all verification output in story completion notes

## Dev Notes

### Architecture Constraints

- **Module dependency chain (architecture mandatory):** `infra/` → `vault-core/` → `vault-config/` is the ordering enforced via explicit `depends_on`. This is not optional — OpenTofu may parallelize module evaluation without it.
  [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis — Cross-component dependencies]

- **Root outputs are the API:** The root `outputs.tf` is the public contract consumed by downstream initiatives (`terminus-infra-k3s`, `terminus-platform-temporal`, etc.). The architecture specifies required outputs: `vault_addr`, `vault_ca_cert_pem`, `approle_role_id_{service}`. Never remove or rename these without a coordinated migration.
  [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — OpenTofu Outputs Public Contract]

- **NFR14 — Runbooks must be step-by-step:** The Day-0 runbook must be executable by the operator returning to the project after weeks without context. Every prerequisite must be stated. Every command must be shown exactly. "Run tofu apply" is not sufficient — show the full command with env vars.
  [Source: docs/terminus/infra/secrets/prd.md — NFR14]

- **vault-config module stub:** `module "vault-config"` in root `main.tf` should exist as a stub now (empty source module) to establish the `depends_on` contract. Epic 3 stories will add content to it. The stub prevents Epic 3 developers from needing to modify root `main.tf` to establish the dep chain.

- **ca.crt distribution:** `tofu output -raw vault_ca_cert_pem > ca.crt` produces the CA public cert in PEM format. This file should be committed to git at the repository root. Downstream consumers reference it as `VAULT_CACERT=ca.crt` or equivalent. The architecture notes this explicitly for Ansible and OpenTofu provider config.
  [Source: docs/terminus/infra/secrets/architecture.md#TLS & Certificates — D6]

### Project Structure Notes

- This story creates `docs/runbooks/day0-bootstrap.md` (new directory + file).
- Modifies root `main.tf` (finalize module wiring) and root `outputs.tf` (finalize outputs).
- Epic 1 is complete when this story passes. The operator has a running, initialized, unsealed Vault with KV v2 and AppRole mounts active.

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.8]
- [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis]
- [Source: docs/terminus/infra/secrets/architecture.md#Format Patterns — OpenTofu Outputs Public Contract]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
