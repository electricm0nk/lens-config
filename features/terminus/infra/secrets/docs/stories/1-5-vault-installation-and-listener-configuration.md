# Story 1.5: Vault Installation and Listener Configuration

Status: ready-for-dev

## Story

As the operator,
I want a `vault-core/` OpenTofu module that installs Vault 1.21.4 on the provisioned VM and configures its TLS listener and Raft storage backend,
so that the Vault binary is running and reachable before any initialization commands are issued.

## Acceptance Criteria

1. **[Vault installed at pinned version]** Given the `infra/` module has been applied and the VM is running, when `vault-core/` module resources are applied, then Vault version 1.21.4 is installed — verified by `vault version` returning `Vault v1.21.4` exactly (no version float — NFR21).

2. **[TLS listener configured]** Given `vault.hcl` on the VM after apply, when I inspect `/etc/vault.d/vault.hcl`, then:
   - The `listener "tcp"` stanza specifies `address = "0.0.0.0:8200"`, `tls_cert_file`, and `tls_key_file` paths pointing to the server cert and key from infra module outputs
   - `tls_disable = false` (explicit, not omitted)
   - `tls_min_version = "tls12"` (NFR2 — no plaintext HTTP)

3. **[Raft storage configured]** Given `/etc/vault.d/vault.hcl`, when I inspect the `storage "raft"` stanza, then `path = "/opt/vault/data"` and `node_id = "vault-node-1"` are set.

4. **[Vault running as systemd service]** Given the VM after apply, when I run `systemctl status vault`, then the service is `active (running)` and `enabled` (starts on reboot).

5. **[Health endpoint reachable — uninitialized state]** Given Vault installed and running (but not yet initialized), when I run `curl -k https://{vm_ip}:8200/v1/sys/health`, then the response is HTTP 501 (not initialized) or HTTP 503 (sealed) — not a connection refused error (FR33).

6. **[Variables wired from infra]** Given `modules/vault-core/variables.tf`, when I inspect it, then it accepts `vault_addr` (string), `ca_cert_pem` (string), `server_cert_pem` (string, sensitive), `server_private_key_pem` (string, sensitive), and `vm_ip` (string) from infra module outputs.

## Tasks / Subtasks

- [ ] **Task 1: Implement Vault installation in `modules/vault-core/main.tf`** (AC: 1, 4)
  - [ ] Use `null_resource` with `remote-exec` provisioner to install Vault on the VM:
    - Install via HashiCorp package repo (or direct binary download) at exact version `1.21.4`
    - Create vault service user and group (`vault:vault`)
    - Create directories: `/etc/vault.d/`, `/opt/vault/data/` (owned by vault user)
    - Enable and start vault systemd service after configuration is in place
  - [ ] Alternatively, use Ansible via `local-exec` calling `ansible-playbook` (consistent with Day-2 pattern) — either approach is acceptable; OpenTofu `null_resource` is the architecture-specified mechanism for vault-core init

- [ ] **Task 2: Generate and place vault.hcl via null_resource** (AC: 2, 3)
  - [ ] Template `vault.hcl` with Raft storage stanza:
    ```hcl
    storage "raft" {
      path    = "/opt/vault/data"
      node_id = "vault-node-1"
    }
    listener "tcp" {
      address       = "0.0.0.0:8200"
      tls_cert_file = "/etc/vault.d/server.crt"
      tls_key_file  = "/etc/vault.d/server.key"
      tls_min_version = "tls12"
    }
    api_addr     = "https://${var.vm_ip}:8200"
    cluster_addr = "https://${var.vm_ip}:8201"
    ui           = false
    ```
  - [ ] Write server cert (`var.server_cert_pem`) and server key (`var.server_private_key_pem`) to `/etc/vault.d/server.crt` and `/etc/vault.d/server.key` on the VM
  - [ ] Set file permissions: `server.key` owned by vault user, `chmod 0640`; `server.crt` owned by vault, `chmod 0644`

- [ ] **Task 3: Implement `modules/vault-core/variables.tf`** (AC: 6)
  - [ ] Declare all variables: `vault_addr`, `ca_cert_pem`, `server_cert_pem` (sensitive), `server_private_key_pem` (sensitive), `vm_ip`
  - [ ] Declare `vault_version` (string) — used to pin the exact install command version string

- [ ] **Task 4: Add initial stub to `modules/vault-core/outputs.tf`** (for Story 1.6 wiring)
  - [ ] Add output `vault_running = true` as a static value (used as a signal for Story 1.6 null_resource `depends_on`)
  - [ ] Note with `# TODO: vault_initialized added in Story 1.6` comment

- [ ] **Task 5: Wire `module "vault-core"` in root `main.tf`** (AC: 1, 6)
  - [ ] Add `module "vault-core"` block with `depends_on = [module.infra]`
  - [ ] Pass all required variables from infra module outputs:
    - `vm_ip = module.infra.vm_ip`
    - `server_cert_pem = module.infra.server_cert_pem`
    - `server_private_key_pem = module.infra.server_private_key_pem`
    - `ca_cert_pem = module.infra.ca_cert_pem`
    - `vault_addr = "https://${module.infra.vm_ip}:8200"`
  - [ ] Pass `providers = { vault = vault.bootstrap }` — vault provider alias established in Story 1.6; for this story, stub the provider pass-through or leave as a comment marking where it will be added

- [ ] **Task 6: Verify health endpoint after apply** (AC: 5)
  - [ ] After apply, test: `curl -k https://{vm_ip}:8200/v1/sys/health`
  - [ ] Confirm HTTP 501 (uninitialized) response — not 000 (connection refused)
  - [ ] Document in story completion notes: Vault version output, systemd status output

## Dev Notes

### Architecture Constraints

- **Toolchain version pin (NFR21, NFR22):** Vault 1.21.4 must be installed exactly — not `>= 1.21`, not latest. The `vault_version` variable default is `"1.21.4"`. Upgrade procedure: update default value → `tofu plan` → review → `tofu apply`.
  [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints — Toolchain]

- **`null_resource` with `remote-exec` is the specified approach (D2 context):** The architecture decision for D2 notes that `null_resource` with `remote-exec` is used for vault init. The same mechanism applies to installing the Vault binary and configuring the service. The remote-exec provisioner requires SSH connectivity to the VM.
  [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2]

- **`ui = false` in vault.hcl:** The Vault UI is explicitly out of scope (attack surface reduction). Do NOT set `ui = true`.
  [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints — Non-negotiable from PRD]

- **TLS minimum version:** `tls_min_version = "tls12"` is required. NFR2 mandates TLS everywhere; TLS 1.0/1.1 are deprecated and must not be accepted.
  [Source: docs/terminus/infra/secrets/prd.md — NFR2]

- **Port 8201:** `cluster_addr` port 8201 must be declared in `vault.hcl` even for single-node. Raft requires it for the cluster communication path even without peers.

- **depends_on ordering:** `module "vault-core"` MUST have `depends_on = [module.infra]` in root `main.tf`. This ensures the VM exists and TLS certs are available before vault-core provisioner runs. This is the explicit dependency chain from the architecture.
  [Source: docs/terminus/infra/secrets/architecture.md#Decision Impact Analysis — Cross-component dependencies]

### Project Structure Notes

- This story adds content to `modules/vault-core/main.tf`, `modules/vault-core/variables.tf`, and `modules/vault-core/outputs.tf` which are stubs from Story 1.1.
- New files written ON the VM (not in the repo): `/etc/vault.d/vault.hcl`, `/etc/vault.d/server.crt`, `/etc/vault.d/server.key`
- No new OpenTofu files at repo root are created by this story (root module wiring is finalized in Story 1.8).

### References

- [Source: docs/terminus/infra/secrets/epics.md#Story 1.5]
- [Source: docs/terminus/infra/secrets/architecture.md#Bootstrap & Credential Delivery — D2]
- [Source: docs/terminus/infra/secrets/architecture.md#Complete Project Directory Structure]
- [Source: docs/terminus/infra/secrets/architecture.md#Technical Constraints — Toolchain]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
