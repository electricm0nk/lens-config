# Story 1.2: Proxmox VM Provisioning Module

Status: ready-for-dev

## Story

As the operator,
I want an `infra/` OpenTofu module that provisions a Proxmox VM with defined networking,
so that the Vault server has a reproducible infrastructure layer that can be fully destroyed and reprovisioned via `tofu destroy -target module.infra && tofu apply`.

## Acceptance Criteria

1. **[VM provisioning]** Given a Proxmox environment with the proxmox provider configured, when I run `tofu apply -target module.infra`, then a VM is created matching the specified `proxmox_node` and VM configuration, and `tofu plan -target module.infra` shows no changes after a clean apply (NFR16).

2. **[Static IP output]** Given the `infra/` module, when I inspect `modules/infra/outputs.tf`, then `vm_ip` is declared as a string output containing the static IP address assigned to the VM.

3. **[Clean destroy]** Given a provisioned VM, when I run `tofu destroy -target module.infra`, then the VM is removed from Proxmox with exit code 0 and no dangling resources.

4. **[Provider isolation]** Given `modules/infra/main.tf`, when I inspect all provider references, then only `hashicorp/proxmox` and `hashicorp/tls` providers are referenced (no `hashicorp/vault` or `carlpett/sops` calls — no cross-layer provider bleed).

5. **[All Proxmox config contained]** Given all OpenTofu module files, when I grep for `proxmox` across all `modules/vault-core/` and `modules/vault-config/` files, then zero matches are returned — all Proxmox-specific resources are exclusively within `modules/infra/`.

6. **[Variables wired]** Given `modules/infra/variables.tf`, when I inspect it, then it accepts `proxmox_node` (string), `vault_version` (string), `vm_name` (string, default `"vault-vm"`), and `vm_ip` (string — static IP to assign) as inputs.

## Tasks / Subtasks

- [ ] **Task 1: Implement `modules/infra/main.tf`** (AC: 1, 4, 5)
  - [ ] Add `proxmox_virtual_environment_vm` resource (or equivalent `proxmox_vm_qemu` depending on provider version) for the Vault VM
  - [ ] Configure VM: `proxmox_node`, CPU/memory appropriate for Vault (2 vCPU, 2GB minimum), disk size (20GB minimum for Raft data), static IP via DHCP reservation or `ipconfig0` stanza
  - [ ] Set `clone` or `iso` source appropriate to the environment — use a variable for the template/image so it is not hardcoded
  - [ ] Confirm provider reference is `hashicorp/proxmox` only — no `vault` or `sops` resources in this file

- [ ] **Task 2: Implement `modules/infra/variables.tf`** (AC: 6)
  - [ ] Declare `proxmox_node` (string)
  - [ ] Declare `vault_version` (string) — used to label the VM desc/note for traceability
  - [ ] Declare `vm_name` (string, default `"vault-vm"`)
  - [ ] Declare `vm_ip` (string) — static IP address to assign; required, no default
  - [ ] Declare `vm_template` or `vm_clone_source` (string) — the Proxmox template/clone source; required, no default

- [ ] **Task 3: Implement `modules/infra/outputs.tf`** (AC: 2)
  - [ ] Declare `vm_ip` (string) — the static IP address; used by root module to pass to `vault-core/` and for `vault_addr` construction
  - [ ] Add `description` field to all outputs (architecture contract: no silent outputs)

- [ ] **Task 4: Wire `module "infra"` in root `main.tf`** (AC: 1, 6)
  - [ ] Pass `proxmox_node = var.proxmox_node`, `vault_version = var.vault_version`, `vm_ip = var.vm_ip` (add `vm_ip` to root `variables.tf` if not present)
  - [ ] Pass `providers = { proxmox = proxmox }` explicitly (provider isolation pattern)
  - [ ] Update root `outputs.tf` stub: wire `vault_addr = "https://${module.infra.vm_ip}:8200"` (replaces TODO comment from Story 1.1)

- [ ] **Task 5: Verify idempotency** (AC: 1, NFR16)
  - [ ] After a successful `tofu apply -target module.infra`, run `tofu plan -target module.infra`
  - [ ] Confirm output shows "No changes. Your infrastructure matches the configuration."
  - [ ] Document any known non-idempotent behaviors (e.g., cloud-init runs) and how to handle them

## Dev Notes

### Architecture Constraints

- **Provider isolation is mandatory (architecture pattern):** `hashicorp/proxmox` is used ONLY within `modules/infra/`. It is declared in the root `main.tf` provider block but passed to this module via the `providers` argument. Never re-declare the proxmox provider within `modules/vault-core/` or `modules/vault-config/`.
  [Source: docs/terminus/infra/secrets/architecture.md#Implementation Patterns — Structure Patterns]

- **Layered module rationale:** The `infra/` module must be independently targetable (`-target module.infra`) so that blast-and-repave can destroy and reprovision the VM layer without touching Vault config. This is the primary reason for module separation (D2 decision context).
  [Source: docs/terminus/infra/secrets/architecture.md#Starter Approach — Selected Approach: Layered OpenTofu Modules]

- **Static IP required:** Vault's TLS cert (Story 1.3) is generated against the VM's IP address. Dynamic IP assignment (DHCP without reservation) would invalidate the TLS cert on every reprovision. Assign a static IP via Proxmox VM `ipconfig0` or a DHCP reservation — the IP must be stable across destroy/apply cycles.
  [Source: docs/terminus/infra/secrets/architecture.md#Core Architectural Decisions — D6]

- **vm_ip as root variable:** `vm_ip` must be declared in root `variables.tf` so it can be provided at apply time. It flows: root `var.vm_ip` → `module.infra` → `module.infra.vm_ip` output → root `outputs.vault_addr`. This is not auto-detected from Proxmox — the operator specifies the static IP.

- **Proxmox provider version:** Pin the Proxmox provider in the root `versions.tf` `required_providers` block (established in Story 1.1). Confirm the provider version used here is compatible with the Proxmox environment version and matches the lock file pin.

### Project Structure Notes

- All `modules/infra/` files are already created as stubs from Story 1.1. This story replaces stub content with real implementations.
- Do NOT create any files outside `modules/infra/` except updating root `main.tf` (module call) and root `variables.tf` (add `vm_ip` if missing) and root `outputs.tf` (wire `vault_addr`).
- `modules/infra/` does NOT contain a `policies/` subdirectory — that is `modules/vault-config/policies/` only.

### OpenTofu Resource Naming Rule

Resource name follows architecture pattern `{resource_type}.{service}`:
- VM resource: `proxmox_virtual_environment_vm.vault` or `proxmox_vm_qemu.vault`
- No adjectives or suffixes unless disambiguating
  [Source: docs/terminus/infra/secrets/architecture.md#Naming Patterns — OpenTofu Resource Naming]

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.2]
- [Source: docs/terminus/infra/secrets/architecture.md#Starter Approach — Layered OpenTofu Modules]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]
- [Source: docs/terminus/infra/secrets/architecture.md#Implementation Patterns — Naming Patterns]
- [Source: docs/terminus/infra/secrets/architecture.md#Implementation Patterns — Format Patterns — OpenTofu Outputs]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
