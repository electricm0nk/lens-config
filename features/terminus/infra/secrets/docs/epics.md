---
stepsCompleted: [step-01-validate-prerequisites, step-02-design-epics, step-03-create-stories, step-04-final-validation]
inputDocuments:
  - docs/terminus/infra/secrets/prd.md
  - docs/terminus/infra/secrets/architecture.md
  - docs/terminus/infra/secrets/adversarial-review-report.md
workflowType: 'epics-and-stories'
initiative: terminus-infra-secrets
generatedAt: 2026-03-24T01:30:00Z
mode: batch
---

# terminus-infra-secrets — Epic & Story Breakdown

## Overview

This document decomposes the `terminus-infra-secrets` requirements into implementable epics and stories. The initiative bootstraps a Vault KV v2 + SOPS+age secret management capability for the terminus homelab. All 39 FRs and 23 NFRs are covered. Nine mandatory advisory review items from the adversarial-review-report.md are incorporated as explicit stories (marked **[AR]**).

**Target repo:** `terminus.infra`
**Output path:** `docs/terminus/infra/secrets/`
**Operator persona:** Single operator — Todd (homelab, financial-sector-bar discipline)

---

## Requirements Inventory

### Functional Requirements

FR1: Operator can store an encrypted secret in git using SOPS+age without plaintext ever touching the repository
FR2: Operator can sync a SOPS-encrypted secret from git into Vault KV v2 at a defined path hierarchy
FR3: Authorized workload can retrieve a secret from Vault KV v2 using its AppRole identity
FR4: Operator can retrieve a specific version of a secret from Vault KV v2 (version history)
FR5: Operator can soft-delete a secret from Vault KV v2 without destroying version history
FR6: Operator can provision a new AppRole identity for a workload via OpenTofu (policy-as-code)
FR7: Operator can scope an AppRole identity to a minimum set of Vault paths via a HCL policy committed to git
FR8: Operator can revoke an AppRole secret-id without affecting other workload identities
FR9: Operator can deliver an AppRole secret-id to a workload via wrapped token (single-use delivery)
FR10: System enforces default-deny — any path not explicitly granted in policy returns 403
FR11: Operator can verify that a workload identity cannot read outside its scoped namespace (isolation test)
FR12: Operator can provision a complete, operational Vault instance from zero via a single `tofu apply`
FR13: Operator can reprovision Vault entirely from git after full VM destruction (blast-and-repave) with no manual steps
FR14: Operator can apply Vault configuration changes via `tofu apply` without manual Vault CLI interaction
FR15: Operator can run Ansible playbooks for Day-2 operations from git
FR16: OpenTofu can retrieve secrets from Vault during infrastructure provisioning runs using its AppRole identity
FR17: System records every secret read, write, auth attempt, and denial to a structured audit log
FR18: Operator can verify audit log contains an entry for a specific operation within 30 seconds
FR19: Operator can confirm audit log resumes after a Vault restart or restore
FR20: Operator can produce a record of all privileged operations from the audit log
FR21: Operator can document a known compliance deviation with compensating controls in a formal deviation record
FR22: Operator can define a TTL for a secret or AppRole credential (maximum 90 days)
FR23: Operator can execute a manual secret rotation end-to-end
FR24: System can automatically rotate a secret before its TTL expires (automated rotation — Growth; hook designed in MVP)
FR25: Operator can document and execute an age key rotation, re-encrypting all SOPS-managed files
FR26: Operator can permanently destroy a secret and its SOPS source file per a documented destruction procedure
FR27: Operator can take a Vault Raft snapshot manually on demand
FR28: System takes Vault Raft snapshots automatically on a defined schedule; snapshots stored off-VM
FR29: Operator can restore Vault from a Raft snapshot and verify recovery via three-part acceptance test
FR30: Operator can execute the full blast-and-repave procedure and verify via three-part acceptance test
FR31: Operator can access Vault using the break-glass procedure when normal authentication paths are unavailable
FR32: Operator can rotate and revoke the root token after break-glass use, leaving no persistent root credential
FR33: External tools can query Vault health status via the `/v1/sys/health` endpoint
FR34: Operator can verify Vault seal status at any time via CLI
FR35: Audit log format is structured JSON, compatible with SIEM and Loki log aggregation pipelines
FR36: Consul can perform health checks against the Vault service (mesh integration — Growth)
FR37: Operator can determine currently pinned version of each security-critical component from OpenTofu lock files
FR38: Operator can document a version exception as a comment in the OpenTofu module and in a running exceptions log
FR39: Operator can query CVE databases for known vulnerabilities against pinned component versions (deferred — `terminus-infra-dependency-health`)

### Non-Functional Requirements

NFR1: All secrets at rest encrypted — Vault Raft storage uses AES-256-GCM; SOPS plaintext never persists on disk
NFR2: All secrets in transit encrypted — Vault listener requires TLS; no plaintext HTTP accepted
NFR3: All Vault API authentication uses short-lived tokens derived from AppRole; no long-lived static tokens
NFR4: Vault root token must not persist after bootstrap; any root token use followed by rotation/revocation in same session
NFR5: Every AppRole policy must be reviewable as a committed HCL file in git
NFR6: age private key must never be stored on a network-accessible system unencrypted; custody location documented
NFR7: Vault availability target: best-effort (homelab, single node, planned maintenance acceptable)
NFR8: Vault unavailability must not cause data loss for secrets already stored
NFR9: All downstream workloads must degrade gracefully when Vault is unreachable (consumer responsibility — documented)
NFR10: Vault Raft snapshots must be stored on a different physical host than the Vault VM
NFR11: Vault KV v2 secret read latency: < 2 seconds under normal operating conditions
NFR12: Day-0 bootstrap time (`tofu apply` from zero to operational Vault): ≤ 60 minutes
NFR13: Snapshot restore to operational Vault (RTO): ≤ 4 hours including verification
NFR14: All operational procedures documented as step-by-step runbooks executable without prior context
NFR15: No manual Vault configuration permitted outside documented break-glass scenarios
NFR16: `tofu plan` outputting no changes is the correct steady state
NFR17: All operational runbooks stored in git alongside the code they operate
NFR18: Blast-and-repave executable as a single `tofu apply` with no manual steps beyond providing the age key
NFR19: Snapshot restore procedure documented, tested at least once, producing verified RTO measurement
NFR20: Recovery procedures tested on a defined schedule — not assumed to work without verification
NFR21: Vault and all security-critical components must have pinned versions in OpenTofu lock files
NFR22: Component upgrades applied via `tofu plan` → review → `tofu apply`
NFR23: All version exceptions documented with reason and target resolution version

### Additional Requirements (from Architecture)

- Module layer separation is mandatory: `infra/` → `vault-core/` → `vault-config/` with enforced dependency ordering via `depends_on` in root module
- Vault provider alias pattern required for root token handoff during `vault-core/` initialization (HIGH complexity — S1 from adversarial review)
- SOPS state encryption discipline: `terraform.tfstate` must be SOPS-encrypted before every `git add`; git pre-commit hook required to block accidental unencrypted state commit (S3 from adversarial review)
- TLS CA cert committed unencrypted to git; client trust configuration (`VAULT_CACERT` for Ansible, `ca_cert_pem` in Vault provider config) must be explicitly automated (S2 from adversarial review)
- Path hierarchy `secret/terminus/{domain}/{service}/{key}` is a stable, breaking-change-sensitive contract — never rename silently
- Three workload AppRole identities are all required in MVP (not just one proof of concept): opentofu, k3s, openclaw
- Isolation verification suite (verify-isolation.yml) is a first-class MVP deliverable, not cleanup work (S4 from adversarial review)
- AppRole token TTL strategy must be explicitly configured per workload (S8 from adversarial review)
- Break-glass and blast-and-repave runbooks must include acceptance criteria requiring at least one exercised run (S5, S6 from adversarial review)
- Automated rotation Growth-phase hook must be designed in MVP even if not implemented (S9 from adversarial review)
- Toolchain versions pinned: Vault 1.21.4 | OpenTofu 1.11.0 | Consul 1.22.5 | SOPS + age | Ansible-core 2.19

---

### FR Coverage Map

FR1: Epic 2 — SOPS secret authoring (encrypt without plaintext in repo)
FR2: Epic 2 — SOPS-to-Vault sync pipeline
FR3: Epic 3 — AppRole secret retrieval
FR4: Epic 2 — KV v2 version history retrieval
FR5: Epic 2 — KV v2 soft-delete
FR6: Epic 3 — AppRole provisioning via OpenTofu
FR7: Epic 3 — HCL policy files committed to git
FR8: Epic 3 — AppRole secret-id revocation
FR9: Epic 3 — Wrapped token delivery
FR10: Epic 3 — Default-deny enforcement
FR11: Epic 3 — Isolation verification suite
FR12: Epic 1 — Single `tofu apply` → operational Vault
FR13: Epic 5 — Blast-and-repave full reprovisioning
FR14: Epic 1 — All Vault config via OpenTofu only
FR15: Epic 4 — Day-2 Ansible playbooks
FR16: Epic 3 — OpenTofu AppRole consumes Vault during provisioning runs
FR17: Epic 3 — Vault audit backend, structured JSON logging
FR18: Epic 3 — Audit log entry verification within 30s
FR19: Epic 5 — Audit log resume after restart/restore
FR20: Epic 3 — Privileged operations record from audit log
FR21: Epic 5 — Compliance deviation records
FR22: Epic 3 — TTL enforcement on secrets and AppRole credentials
FR23: Epic 4 — Manual secret rotation procedure
FR24: Epic 4 — Automated rotation stub/hook (Growth; hook designed in MVP)
FR25: Epic 4 — Age key rotation procedure
FR26: Epic 2 — Secret destruction procedure
FR27: Epic 4 — On-demand manual Raft snapshot
FR28: Epic 4 — Automated scheduled snapshots off-VM
FR29: Epic 5 — Snapshot restore + three-part acceptance test
FR30: Epic 5 — Blast-and-repave + three-part acceptance test
FR31: Epic 5 — Break-glass procedure
FR32: Epic 5 — Root token rotation/revocation after break-glass
FR33: Epic 1 — `/v1/sys/health` endpoint operational after bootstrap
FR34: Epic 1 — `vault status` seal state verification
FR35: Epic 4 — Audit log JSON format, SIEM/Loki-compatible, logrotate configured
FR36: Epic 4 — Consul health check (Growth; endpoint ready in MVP)
FR37: Epic 5 — Pinned versions readable from lock files
FR38: Epic 5 — Version exception documentation pattern
FR39: Deferred — `terminus-infra-dependency-health` future initiative

---

## Epic List

### Epic 1: Provisioned Vault Instance
Operator can stand up a complete, HA-free, TLS-secured, initialized Vault instance from git via a single `tofu apply`. This epic delivers the load-bearing foundation — without it, no other epic can execute.
**FRs covered:** FR12, FR13 (partial), FR14, FR33, FR34
**NFRs covered:** NFR1, NFR2, NFR4, NFR12, NFR15, NFR16, NFR18, NFR21, NFR22

### Epic 2: Secret Authoring Pipeline
Operator can create, version, sync, rotate, and destroy secrets through the SOPS+age authoring pipeline. Secrets flow from plaintext → SOPS-encrypted in git → Vault KV v2 at the correct path hierarchy, with no plaintext ever touching the repository.
**FRs covered:** FR1, FR2, FR4, FR5, FR25, FR26
**NFRs covered:** NFR1, NFR6, NFR14, NFR17

### Epic 3: Workload Identity Management
Operator can provision, scope, and verify all three required workload identities (opentofu, k3s, openclaw) with per-workload AppRole policies, enforced isolation, audit logging, and an exercised verification suite.
**FRs covered:** FR3, FR6, FR7, FR8, FR9, FR10, FR11, FR16, FR17, FR18, FR20, FR22
**NFRs covered:** NFR3, NFR4, NFR5, NFR11, NFR14, NFR15

### Epic 4: Day-2 Operations
Operator can perform all ongoing maintenance operations — secret rotation, snapshot management, audit log rotation, and certificate renewal — through documented, tested Ansible playbooks without any manual Vault CLI interaction.
**FRs covered:** FR15, FR23, FR24 (hook), FR27, FR28, FR35, FR36 (stub)
**NFRs covered:** NFR10, NFR14, NFR15, NFR17, NFR20

### Epic 5: Disaster Recovery & Resilience
Operator can recover from any failure scenario — full VM destruction via blast-and-repave, data corruption via snapshot restore, and authentication path failure via break-glass — with documented, tested, RTO-measured procedures.
**FRs covered:** FR13, FR19, FR21, FR29, FR30, FR31, FR32, FR37, FR38
**NFRs covered:** NFR13, NFR18, NFR19, NFR20, NFR21, NFR22, NFR23

---

## Epic 1: Provisioned Vault Instance

**Goal:** Operator can execute `tofu apply` from a clean environment and have a running, TLS-secured, initialized Vault instance provisioned on Proxmox. The single-apply goal (NFR18) means every resource from VM creation through Vault initialization to root token revocation is orchestrated in a deterministic sequence with no manual intervention.

### Story 1.1: Repository Skeleton and Provider Configuration

As the operator,
I want a repository structure with all required OpenTofu provider configurations, version pins, and a .gitignore that prevents accidental secrets commit,
So that I have a reproducible, safe foundation for all subsequent OpenTofu work.

**Acceptance Criteria:**

**Given** a clean `terminus.infra` repository,
**When** I inspect the root of the repository,
**Then** `main.tf`, `variables.tf`, `outputs.tf`, and `Makefile` exist at the root,
**And** `.gitignore` excludes `.terraform/`, `*.tfstate.bak`, `*.tfstate` (unencrypted), plaintext key files (*.age, *.pem outside committed paths)
**And** `.terraform.lock.hcl` is committed and pins all provider versions: `hashicorp/proxmox` (latest compatible), `hashicorp/vault` (matching Vault 1.21.4), `hashicorp/tls`, `carlpett/sops` (for SOPS secret decryption in vault-config)
**And** `variables.tf` declares `proxmox_node`, `vault_version` (default: `"1.21.4"`), `snapshot_host`, `sops_age_key_file`
**And** `tofu init` completes with exit code 0 and no provider version drift warnings

**Given** the Makefile,
**When** I run `make help`,
**Then** at minimum `plan`, `apply`, `encrypt-state`, `verify` targets are listed with descriptions

---

### Story 1.2: Proxmox VM Provisioning Module

As the operator,
I want an `infra/` OpenTofu module that provisions a Proxmox VM with defined networking,
So that the Vault server has a reproducible infrastructure layer that can be fully destroyed and reprovisioned via `tofu destroy -target module.infra && tofu apply`.

**Acceptance Criteria:**

**Given** a Proxmox environment with the proxmox provider configured,
**When** I run `tofu apply -target module.infra`,
**Then** a VM is created matching the specified `proxmox_node` and VM configuration
**And** the VM has a static IP address output as `vm_ip` from `modules/infra/outputs.tf`
**And** `tofu plan -target module.infra` shows no changes after a clean apply (NFR16)
**And** `tofu destroy -target module.infra` cleanly removes the VM with exit code 0

**Given** the infra module,
**When** I inspect `modules/infra/main.tf`,
**Then** only the `hashicorp/proxmox` and `hashicorp/tls` providers are used (no cross-layer provider bleed)
**And** all Proxmox-specific configuration is contained within `modules/infra/` only

---

### Story 1.3: TLS Certificate Generation (D6)

As the operator,
I want the `infra/` module to generate a self-signed CA and server certificate for Vault's TLS listener,
So that all Vault API traffic is encrypted from the first `tofu apply` with no manual cert management.

**Acceptance Criteria:**

**Given** `modules/infra/main.tf` with the `hashicorp/tls` provider,
**When** I run `tofu apply -target module.infra`,
**Then** a self-signed CA private key and certificate are generated
**And** a Vault server certificate signed by the CA is generated for the VM's IP
**And** `modules/infra/outputs.tf` exposes `ca_cert_pem` (non-sensitive), `ca_private_key_pem` (sensitive), and `server_cert_pem` (non-sensitive), `server_private_key_pem` (sensitive)
**And** `ca_cert_pem` and `server_cert_pem` are NOT marked sensitive (they are public trust anchors)
**And** `ca_private_key_pem` and `server_private_key_pem` ARE marked `sensitive = true`
**And** the CA cert will be committed to git unencrypted as a consumer trust anchor (verified by presence in `outputs.tf` description)

---

### Story 1.4: SOPS State Encryption Guard [AR-S3 HIGH]

As the operator,
I want a Makefile `encrypt-state` target and a git pre-commit hook that prevents committing unencrypted `terraform.tfstate`,
So that sensitive state values (AppRole IDs, root tokens, TLS private keys) can never be accidentally exposed in the repository.

**Acceptance Criteria:**

**Given** the repository with SOPS+age configured,
**When** I run `make encrypt-state`,
**Then** `terraform.tfstate` is SOPS-encrypted in-place using the configured age recipient
**And** the encrypted file is committed-safe (contains SOPS metadata header, no plaintext sensitive values)
**And** the command exits 0 on success and non-zero if encryption fails

**Given** the repository has a `.git/hooks/pre-commit` hook installed,
**When** I run `git add terraform.tfstate` where the file is NOT SOPS-encrypted,
**Then** the pre-commit hook detects the unencrypted state (checks for absence of SOPS metadata)
**And** the commit is blocked with a clear error: `ERROR: terraform.tfstate appears unencrypted. Run 'make encrypt-state' first.`

**Given** a correctly SOPS-encrypted `terraform.tfstate`,
**When** I run `git add terraform.tfstate && git commit`,
**Then** the pre-commit hook passes and the commit succeeds

**Given** the repository README,
**When** I read the Development Setup section,
**Then** instructions for installing the pre-commit hook are documented (e.g., `make install-hooks`)

---

### Story 1.5: Vault Installation and Listener Configuration

As the operator,
I want a `vault-core/` OpenTofu module that installs Vault 1.21.4 on the provisioned VM and configures its TLS listener and Raft storage backend,
So that the Vault binary is running and reachable before any initialization commands are issued.

**Acceptance Criteria:**

**Given** the `infra/` module has been applied and `vm_ip` + TLS certs are available,
**When** I run `tofu apply -target module.vault-core` (with vault-core receiving infra outputs),
**Then** Vault 1.21.4 is installed at version-pinned exactly (no float)
**And** `vault.hcl` configures a TLS listener on port 8200 referencing the server cert and key from infra outputs
**And** the storage stanza uses `raft` with `path = "/opt/vault/data"`
**And** Vault is started as a systemd service and is in `active (running)` state
**And** `curl -k https://{vm_ip}:8200/v1/sys/health` returns HTTP 501 (uninitialized) or HTTP 503 (sealed) — not connection refused (FR33)
**And** `modules/vault-core/variables.tf` accepts `vault_addr`, `ca_cert_pem`, `server_cert_pem`, `server_private_key_pem` from infra module outputs

---

### Story 1.6: Vault Initialization with Provider Alias Pattern [AR-S1 HIGH]

As the operator,
I want the `vault-core/` module to initialize Vault via `vault operator init`, capture the unseal key and root token, and configure the Vault provider alias so that subsequent OpenTofu resources authenticate correctly — all within a single `tofu apply`,
So that the root token bootstrapping sequence is fully automated with no manual CLI steps required.

**Acceptance Criteria:**

**Given** Vault is installed and running (Story 1.5 complete),
**When** I run `tofu apply` and vault-core is applied,
**Then** a `null_resource` with a `remote-exec` provisioner executes `vault operator init -key-shares=1 -key-threshold=1`
**And** the unseal key and root token are captured from the init output and passed as `sensitive` OpenTofu local values
**And** a `null_resource` immediately executes `vault operator unseal <unseal_key>` before any Vault provider resources are created
**And** the root module declares a `provider "vault" { alias = "bootstrap" }` configured with the init-captured root token
**And** this alias is passed to `module "vault-core"` via the `providers` argument — never the default provider
**And** the LAST resource in `vault-core/` is a `null_resource` that executes `vault token revoke <root_token>` (satisfying NFR4)
**And** `tofu apply` completes with exit code 0 and a running, unsealed, root-token-revoked Vault
**And** `vault status` returns `Sealed = false` and `Initialized = true`

**Given** a junior implementer reading `vault-core/main.tf`,
**When** they review the provider alias pattern,
**Then** an inline comment explains: `# root_token is captured from vault operator init stdout via null_resource.vault_init`
**And** the comment notes: `# the vault.bootstrap alias exists ONLY during vault-core provisioning and is revoked as the final step`

---

### Story 1.7: KV v2 Mount and AppRole Auth Backend

As the operator,
I want the `vault-core/` module to enable the KV v2 secrets engine at `secret/` and enable the AppRole auth backend,
So that the mount points are ready before `vault-config/` provisions policies and identities.

**Acceptance Criteria:**

**Given** Vault initialized and unsealed with root token available to provider alias,
**When** `vault-core/` resources are applied,
**Then** `vault secrets list` includes `secret/` with type `kv` (version 2)
**And** `vault secrets get secret/` shows `options.version = 2`
**And** `vault auth list` includes `approle/` with type `approle`
**And** `tofu plan` on vault-core shows no changes after a clean apply (NFR16)
**And** `vault-core/outputs.tf` exposes `vault_initialized = true` as a boolean signal for root module `depends_on`

---

### Story 1.8: Root Module Wiring and Day-0 Bootstrap Runbook

As the operator,
I want the root `main.tf` to wire all three modules in correct dependency order with explicit `depends_on`, expose the required public contract outputs, and have a complete Day-0 runbook,
So that `tofu apply` from zero executes the full provisioning sequence, and a returning operator can execute Day-0 from documentation alone.

**Acceptance Criteria:**

**Given** the root `main.tf`,
**When** I inspect the module declarations,
**Then** `module "vault-core"` has `depends_on = [module.infra]`
**And** `module "vault-config"` has `depends_on = [module.vault-core]`
**And** `outputs.tf` exposes `vault_addr` (`https://{vm_ip}:8200`), `vault_ca_cert_pem` (non-sensitive), and `approle_role_id_{service}` outputs for each registered workload
**And** all sensitive outputs are marked `sensitive = true`

**Given** `docs/runbooks/day0-bootstrap.md`,
**When** I read it having never seen the repository before,
**Then** the runbook covers: prerequisites (age key location, Proxmox credentials), environment variable setup (`SOPS_AGE_KEY_FILE`, `TF_VAR_proxmox_*`, `TF_VAR_vault_version`), the full `tofu apply` command sequence, operator obligation (capture unseal key from stdout), and post-apply verification steps
**And** the runbook includes a "What to verify after bootstrap" section with: `vault status`, `curl /v1/sys/health`, and a note to run `ansible-playbook verify-isolation.yml` after vault-config is applied

**Given** a fully applied root module,
**When** I run `tofu plan`,
**Then** the output shows "No changes" (NFR16)

---

## Epic 2: Secret Authoring Pipeline

**Goal:** Operator can create, version, sync, and destroy secrets using the SOPS+age pipeline. Secrets flow from plaintext → SOPS-encrypted file in git → Vault KV v2 at the correct path hierarchy, with no plaintext persisting on disk or in the repository at any point.

### Story 2.1: SOPS Configuration and Age Key Setup

As the operator,
I want a `.sops.yaml` configuration file that specifies the age public key as the SOPS recipient for all `sops/*.sops.yaml` files,
So that any secret file created under `sops/` is automatically encrypted to the correct recipient without requiring per-file configuration.

**Acceptance Criteria:**

**Given** the terminus.infra repository,
**When** I inspect `sops/.sops.yaml`,
**Then** the file specifies the age public key (`age1...`) as the sole creation rule recipient
**And** the path rule matches `*.sops.yaml` files under the `sops/` directory
**And** `sops/` directory contains a `.gitkeep` or initial empty file so the directory is tracked

**Given** `docs/runbooks/age-key-custody.md`,
**When** I read it,
**Then** the runbook documents: where the age private key is stored (offline media, not network-accessible — NFR6), how to verify the key is the correct recipient, and the custody transfer procedure if the key is compromised
**And** the runbook explains what happens if the age key is lost (all SOPS-encrypted files become unrecoverable) and the recovery path (restore from Vault Raft snapshot if state was committed and key is gone)

---

### Story 2.2: Bootstrap Credentials Management

As the operator,
I want a `sops/bootstrapcreds.sops.yaml` file that holds the bootstrap-phase credentials needed before Vault exists,
So that the Day-0 bootstrap process has a documented, encrypted location for ephemeral credentials that must not persist beyond their use.

**Acceptance Criteria:**

**Given** SOPS configured with age recipient (Story 2.1),
**When** I run `sops sops/bootstrapcreds.sops.yaml` and inspect the decrypted contents,
**Then** the file contains documented comment headers describing what each key is for
**And** the plaintext comment at the top of the encrypted file documents the key names: `# Keys: (list any ephemeral bootstrap credentials stored here)`
**And** the commit message for this file follows the `chore(sops): add bootstrap credentials template` convention

**Given** the Day-0 runbook,
**When** I consult `docs/runbooks/age-key-custody.md` about bootstrapcreds,
**Then** it is documented that `bootstrapcreds.sops.yaml` contents are ephemeral and should be cleared or rotated after a successful Day-0 bootstrap

---

### Story 2.3: Secret Authoring Workflow (FR1, FR4, FR5)

As the operator,
I want to create, version, and soft-delete secrets in Vault KV v2 via a documented SOPS-to-Vault authoring workflow,
So that all secret material is encrypted at rest in git and versioned in Vault with no plaintext exposure.

**Acceptance Criteria:**

**Given** a plaintext secret value (e.g., `postgres app_role_password`),
**When** I follow the secret authoring workflow (create `sops/postgres.sops.yaml`, encrypt with `sops -e`),
**Then** the resulting file in git contains no plaintext credentials (SOPS metadata header present, all values encrypted)
**And** `sops -d sops/postgres.sops.yaml` decrypts successfully with the age key

**Given** an encrypted `sops/postgres.sops.yaml` synced to Vault,
**When** I run `vault kv get secret/terminus/infra/postgres/app_role_password`,
**Then** the current version is returned with the correct value
**And** the path follows the `secret/terminus/{domain}/{service}/{key}` hierarchy exactly

**Given** a Vault KV v2 secret at a defined path,
**When** I run `vault kv get -version=1 secret/terminus/infra/postgres/app_role_password`,
**Then** the previous version is returned (FR4 — version history)

**Given** a Vault KV v2 secret,
**When** I run `vault kv delete secret/terminus/infra/postgres/app_role_password`,
**Then** the secret is soft-deleted (cannot be read without version flag) but version history is preserved (FR5)
**And** `vault kv get -version=1` still returns the historical value

**Given** `docs/runbooks/secret-authoring.md`,
**When** I read the runbook,
**Then** it covers the complete workflow: `sops -e` → git commit → sync to Vault KV v2 → verify read → optional soft-delete

---

### Story 2.4: SOPS-to-Vault Sync Pipeline (FR2, FR3)

As the operator,
I want an explicit, documented sync step that reads SOPS-encrypted files and writes their values into Vault KV v2 at the correct path,
So that the git source of truth and Vault runtime state are kept in sync via a repeatable, auditable operation.

**Acceptance Criteria:**

**Given** a valid `sops/postgres.sops.yaml` encrypted with the age key,
**When** the OpenTofu `vault-config/` module is applied (with `SOPS_AGE_KEY_FILE` set),
**Then** the secret value is written to Vault KV v2 at `secret/terminus/infra/postgres/app_role_password`
**And** the OpenTofu resource uses `data "sops_file"` or equivalent provider to decrypt transiently — no plaintext written to disk or terraform state in readable form

**Given** a workload's AppRole identity (Story 3.x),
**When** the workload authenticates and requests `vault kv get secret/terminus/infra/postgres/app_role_password`,
**Then** Vault returns the secret value and the audit log records the read (FR3, FR17)
**And** read latency is < 2 seconds under normal conditions (NFR11)

---

### Story 2.5: Secret Destruction Procedure (FR26)

As the operator,
I want a documented, tested procedure for permanently destroying a secret — removing it from both Vault KV v2 and its SOPS source file in git,
So that secret destruction is a deliberate, auditable operation that eliminates all copies.

**Acceptance Criteria:**

**Given** a secret at `secret/terminus/infra/postgres/app_role_password` in Vault and `sops/postgres.sops.yaml` in git,
**When** I follow the destruction procedure,
**Then** `vault kv destroy -versions=$(vault kv metadata get -format=json ... | jq .data.current_version) <path>` permanently removes all versions
**And** `vault kv metadata delete <path>` removes the metadata record
**And** `sops/postgres.sops.yaml` is deleted from git and the deletion is committed

**Given** `docs/runbooks/secret-authoring.md` (destruction section),
**When** I execute the destruction procedure and afterward run `vault kv get <path>`,
**Then** Vault returns a 404 with no data (permanently destroyed, not soft-deleted)

---

### Story 2.6: Age Key Rotation (FR25)

As the operator,
I want a documented, tested age key rotation procedure that re-encrypts all SOPS-managed files with a new age key,
So that key rotation is a structured, low-risk operation that leaves no files encrypted to the old key.

**Acceptance Criteria:**

**Given** all `sops/*.sops.yaml` files encrypted with age key A,
**When** I follow the age key rotation procedure in `docs/runbooks/age-key-custody.md`,
**Then** the procedure covers: generate new age key B, update `.sops.yaml` recipient, `sops updatekeys` on every SOPS file, verify decrypt with key B, commit all re-encrypted files, document old key destruction

**Given** the re-encrypted files,
**When** I run `sops -d sops/bootstrapcreds.sops.yaml` with age key A (old key),
**Then** decryption fails with a key-not-found error (old key no longer valid)

**Given** the re-encrypted files with age key B,
**When** I run `sops -d sops/bootstrapcreds.sops.yaml` with age key B,
**Then** decryption succeeds

---

## Epic 3: Workload Identity Management

**Goal:** Operator can provision, scope, and verify all three required workload AppRole identities (opentofu, k3s, openclaw) with per-workload HCL policies, enforced default-deny, configured TTLs, audit logging, and a verification suite that must pass on every reprovisioning.

### Story 3.1: vault-config Module Foundation — Default-Deny and Audit Backend (FR10, FR17, FR18, FR20)

As the operator,
I want the `vault-config/` OpenTofu module to configure the Vault audit backend and enforce its default-deny posture,
So that every Vault operation is logged and no path is accessible without explicit policy grant.

**Acceptance Criteria:**

**Given** the vault-core module is applied (AppRole auth and KV v2 mount active),
**When** `vault-config/` module is applied (using a short-lived token derived from AppRole or via the bootstrap alias),
**Then** `vault audit list` includes a `file` backend writing to `/var/log/vault/audit.log`
**And** the audit log file format is structured JSON (FR35) — verified by `head -1 /var/log/vault/audit.log | python3 -m json.tool` returning a valid JSON object
**And** `vault policy list` includes `default` — and the default policy grants no access to `secret/` paths
**And** attempting to read `vault kv get secret/terminus/infra/postgres/app_role_password` with only the `default` policy returns HTTP 403 (FR10)

**Given** the audit log is active,
**When** I perform a secret read operation,
**Then** within 30 seconds an audit entry appears in `/var/log/vault/audit.log` with the path, operation, and requester identity (FR18)
**And** the entry's `type` field is `response` and `auth.display_name` includes the AppRole identity

---

### Story 3.2: AppRole Token TTL Policy [AR-S8 MEDIUM]

As the operator,
I want a defined token TTL strategy for each AppRole identity configured in `vault-config/` via OpenTofu,
So that AppRole-derived tokens have intentional lifetimes that balance workload availability against credential exposure.

**Acceptance Criteria:**

**Given** `modules/vault-config/main.tf`,
**When** I review the AppRole role configuration for each workload,
**Then** each `vault_approle_auth_backend_role` resource explicitly sets `token_ttl` and `token_max_ttl`
**And** `token_max_ttl` is ≤ 90 days (NFR3 constraint — no long-lived tokens)
**And** the TTL values for each workload are documented with rationale in `docs/runbooks/workload-onboarding.md`

**Given** `docs/runbooks/workload-onboarding.md`,
**When** I read the AppRole TTL section,
**Then** it explains: short-lived tokens mean a workload must re-authenticate before token expiry; recommended pattern is for workloads to renew tokens before 80% of TTL elapses; OpenTofu and Ansible are operator-triggered (TTL can be longer); continuous workloads like k3s and openclaw require active renewal

**Given** an AppRole token for the `opentofu` workload,
**When** I inspect the token after `vault write auth/approle/login role_id=<id> secret_id=<sid>`,
**Then** `vault token lookup` shows `ttl` matching the configured `token_ttl` for that role

---

### Story 3.3: OpenTofu AppRole Identity and Policy (FR6, FR7, FR16)

As the operator,
I want an `opentofu` AppRole identity with a scoped HCL policy that grants read access to infra secrets and the ability to renew leases,
So that OpenTofu provisioning runs can authenticate to Vault and retrieve secrets without requiring the root token or human-held credentials.

**Acceptance Criteria:**

**Given** `vault-config/policies/opentofu.hcl` committed to git,
**When** I read it,
**Then** it begins with `# Policy for opentofu AppRole — read-only on secret/terminus/infra/*`
**And** it grants `["read", "list"]` on `secret/data/terminus/infra/*` and `secret/metadata/terminus/infra/*`
**And** it grants `["update"]` on `sys/leases/renew`
**And** it denies all other paths (no wildcards beyond the stated scope)

**Given** the policy and AppRole are applied,
**When** I run `vault write auth/approle/login role_id=$OPENTOFU_ROLE_ID secret_id=$OPENTOFU_SECRET_ID`,
**Then** a token is issued and `vault kv get secret/terminus/infra/postgres/app_role_password` succeeds with the opentofu token
**And** `vault kv get secret/terminus/agent/openclaw/api_key` returns HTTP 403 (cross-namespace isolation confirmed)

**Given** `outputs.tf` in `vault-config/`,
**When** I run `tofu output approle_role_id_opentofu`,
**Then** the role ID is returned (and marked sensitive if it contains sensitive values)

---

### Story 3.4: k3s AppRole Identity and Policy (FR6, FR7)

As the operator,
I want a `k3s` AppRole identity with a scoped HCL policy that grants read access only to k3s-scoped secrets,
So that the k3s workload can retrieve its credentials from Vault without access to secrets belonging to other services.

**Acceptance Criteria:**

**Given** `vault-config/policies/k3s.hcl` committed to git,
**When** I read it,
**Then** it begins with `# Policy for k3s AppRole — read-only on secret/terminus/infra/k3s/*`
**And** it grants `["read", "list"]` on `secret/data/terminus/infra/k3s/*` only
**And** any attempt to read `secret/data/terminus/infra/postgres/*` with the k3s token returns HTTP 403

**Given** the k3s AppRole is applied via `tofu apply`,
**When** I authenticate as k3s and attempt cross-workload access,
**Then** k3s can read `secret/terminus/infra/k3s/*` successfully
**And** k3s cannot read `secret/terminus/infra/postgres/*` (403)
**And** k3s cannot read `secret/terminus/agent/openclaw/*` (403)
**And** the audit log records both the successful read and the denied cross-namespace attempt

---

### Story 3.5: OpenClaw AppRole Identity and Isolation Policy (FR6, FR7, FR11)

As the operator,
I want an `openclaw` AppRole identity with a strictly scoped HCL policy that permits access ONLY to OpenClaw's namespace and explicitly denies all other paths,
So that a compromised OpenClaw LLM workload cannot exfiltrate credentials from any other service.

**Acceptance Criteria:**

**Given** `vault-config/policies/openclaw.hcl` committed to git,
**When** I read it,
**Then** it begins with `# Policy for openclaw AppRole — read-only on secret/terminus/agent/openclaw/* — deny all else`
**And** it grants `["read", "list"]` on `secret/data/terminus/agent/openclaw/*`
**And** a Vault `deny` capability is applied to `secret/data/terminus/*` except the openclaw path (or the default-deny posture covers this)

**Given** the openclaw AppRole is applied,
**When** I authenticate as openclaw and attempt reads outside its namespace,
**Then** `vault kv get secret/terminus/infra/postgres/app_role_password` with openclaw token returns HTTP 403
**And** `vault kv get secret/terminus/infra/k3s/credentials` with openclaw token returns HTTP 403
**And** `vault kv get secret/terminus/agent/openclaw/api_key` with openclaw token succeeds
**And** all three outcomes are recorded in the audit log

---

### Story 3.6: Secret-ID Revocation and Wrapped Token Delivery (FR8, FR9)

As the operator,
I want to revoke an AppRole secret-id without affecting other workload identities, and deliver new secret-ids to workloads via wrapped tokens,
So that credential delivery is a single-use, non-replayable operation and revocation is surgical.

**Acceptance Criteria:**

**Given** an active AppRole secret-id for the `k3s` workload,
**When** I run `vault write -f auth/approle/role/k3s/secret-id-accessor/destroy accessor=<accessor>`,
**Then** the specific secret-id is revoked
**And** `vault write auth/approle/login role_id=$K3S_ROLE_ID secret_id=$REVOKED_SECRET_ID` returns an error (revoked)
**And** `vault write auth/approle/login role_id=$OPENTOFU_ROLE_ID secret_id=$OPENTOFU_SECRET_ID` still succeeds (other identities unaffected — FR8)

**Given** a wrapped token request,
**When** I run `vault write -wrap-ttl=60s -f auth/approle/role/k3s/secret-id`,
**Then** a wrapping token is returned (not the actual secret-id)
**And** the wrapping token can only be unwrapped once (FR9 — single-use delivery)

---

### Story 3.7: Isolation Verification Suite [AR-S4 HIGH]

As the operator,
I want an Ansible playbook `ansible/playbooks/verify-isolation.yml` that runs the three-part acceptance test automatically,
So that isolation verification is a first-class, repeatable operation that runs after every reprovisioning — not a one-time manual check.

**Acceptance Criteria:**

**Given** all three workload AppRole identities provisioned (Stories 3.3–3.5),
**When** I run `ansible-playbook ansible/playbooks/verify-isolation.yml`,
**Then** the playbook executes three automated checks:
  1. OpenTofu AppRole can read `secret/terminus/infra/*` — PASS
  2. OpenClaw AppRole receives HTTP 403 on `secret/terminus/infra/*` — PASS
  3. Audit log contains entries for both the permitted read and the denied attempt — PASS
**And** the playbook outputs a clear PASS/FAIL summary per check
**And** any check failure results in a non-zero exit code
**And** the playbook is idempotent — re-running does not change Vault state

**Given** the verify-isolation playbook output,
**When** all three checks pass,
**Then** stdout includes `ISOLATION VERIFICATION: ALL CHECKS PASSED`

**Given** `docs/runbooks/day0-bootstrap.md`,
**When** I read the post-apply verification section,
**Then** it instructs the operator to run `ansible-playbook verify-isolation.yml` as the final step of every bootstrap and blast-and-repave

> **Sprint Planning Note:** Story 3.7 must be scheduled **concurrently with Stories 3.3–3.5**, not sequenced after them. The verify-isolation.yml playbook is built incrementally alongside each AppRole   story — stub the failing checks in 3.3, add k3s check in 3.4, add OpenClaw check in 3.5. Do not defer 3.7 to end of Epic 3 sprint.

---

### Story 3.8: Workload Onboarding End-to-End Operator Journey [AR-S7 MEDIUM]

As the operator,
I want to follow the AppRole onboarding checklist and onboard a new workload end-to-end, verifying the complete journey works without prior context,
So that the onboarding process is repeatable and documented at a level where a returning operator can execute it without assistance.

**Acceptance Criteria:**

**Given** `docs/runbooks/workload-onboarding.md`,
**When** I read it with no prior context about the system,
**Then** the runbook covers Step 1: create `vault-config/policies/{service}.hcl`, Step 2: add `vault_approle_auth_backend_role.{service}` resource, Step 3: add `approle_role_id_{service}` output, Step 4: add isolation test case to `verify-isolation.yml`, Step 5: `tofu plan` → review → `tofu apply`, Step 6: run `verify-isolation.yml` — must pass before workload goes live

**Given** following the runbook to onboard a test workload `test-workload`,
**When** I complete all 6 steps and run `verify-isolation.yml`,
**Then** the test-workload isolation check passes
**And** `tofu plan` shows no changes after the onboarding (NFR16)
**And** the `test-workload` AppRole and policy are then removed (cleanup step in runbook)

---

## Epic 4: Day-2 Operations

**Goal:** Operator can perform all ongoing maintenance — secret rotation, certificate renewal, snapshot management, audit log rotation — through Ansible playbooks that are committed to git and executable without manual Vault CLI interaction.

### Story 4.1: Ansible Control Structure and Inventory

As the operator,
I want an Ansible inventory driven by OpenTofu outputs and an Ansible control structure that establishes the patterns for all Day-2 playbooks,
So that all Ansible operations target the correct Vault VM automatically and operators don't need to manually maintain host lists.

**Acceptance Criteria:**

**Given** a successful `tofu apply` with `vault_addr` and `vm_ip` emitted as outputs,
**When** I inspect `ansible/inventory/hosts.yml`,
**Then** the vault_vm host entry references the IP from `tofu output vm_ip` (either templated or with an explicit instruction to populate from tofu output)
**And** `vault_addr` and `vault_ca_cert_path` are defined as host variables
**And** `ansible -m ping vault_vm` succeeds (reachability verified)

**Given** all Ansible roles,
**When** I inspect any role's `defaults/main.yml`,
**Then** all configurable values (vault_version, snapshot_target_host, snapshot_retention_days) have defaults declared there — no inline defaults in playbooks (convention from architecture naming patterns)

---

### Story 4.2: Vault Audit Log Rotation (D7, FR35, NFR14)

As the operator,
I want the vault-logrotate Ansible role to configure logrotate for Vault's audit log with a 90-day retention policy,
So that the audit log never fills the VM's disk and SIEM-compatible log structure is maintained.

**Acceptance Criteria:**

**Given** the `ansible/roles/vault-logrotate/` role applied via `day0-provision.yml`,
**When** I inspect `/etc/logrotate.d/vault-audit`,
**Then** rotation is set to `daily`
**And** retention is set to `rotate 90` (90 daily logs retained)
**And** `compress` and `delaycompress` are configured
**And** `maxsize 100M` is configured (forced rotation at 100MB threshold)
**And** `postrotate` sends `SIGHUP` to vault process to reopen the log file

**Given** a logrotate dry run,
**When** I run `logrotate -d /etc/logrotate.d/vault-audit`,
**Then** the dry run completes with no errors

**Given** the audit log after rotation,
**When** I inspect `/var/log/vault/`,
**Then** the active log is `audit.log` and rotated logs follow `audit.log.1.gz` naming convention (FR35 — format intact after rotation)

---

### Story 4.3: Vault Raft Snapshot Automation (D5, FR28, NFR10)

As the operator,
I want the vault-snapshot Ansible role to configure an automated Raft snapshot cron job that stores snapshots on a different physical host,
So that a VM failure does not also destroy the snapshot that would enable recovery.

**Acceptance Criteria:**

**Given** the `ansible/roles/vault-snapshot/` role applied via `day0-provision.yml`,
**When** I inspect the cron configuration on the Vault VM,
**Then** a cron job runs `vault operator raft snapshot save /tmp/vault-snapshot.snap && rsync /tmp/vault-snapshot.snap {snapshot_host}:/vault-snapshots/` daily
**And** the cron job runs as the vault service user (not root)
**And** `snapshot_host` is configured in `roles/vault-snapshot/defaults/main.yml` (not hardcoded)

**Given** `snapshot_retention_days = 7` in role defaults,
**When** the snapshot cron runs,
**Then** snapshots older than 7 days on `{snapshot_host}` are pruned

**Given** a monthly archive cron job,
**When** it runs (first of the month),
**Then** a monthly snapshot is taken and retained for 90 days separately (D5 specification)

**Given** the snapshot target host (`{snapshot_host}`),
**When** I verify NFR10,
**Then** `{snapshot_host}` is a different Proxmox host than the Vault VM — documented in `defaults/main.yml` comment

---

### Story 4.4: Manual Secret Rotation Procedure (FR23)

As the operator,
I want an Ansible playbook `day2-rotate-secrets.yml` that executes a manual end-to-end secret rotation,
So that secret rotation is a documented, repeatable procedure that leaves Vault in a clean state with the old credential revoked and the new one active.

**Acceptance Criteria:**

**Given** an active AppRole credential at `secret/terminus/infra/postgres/app_role_password`,
**When** I run `ansible-playbook day2-rotate-secrets.yml -e "target_path=secret/terminus/infra/postgres/app_role_password"`,
**Then** the playbook: (1) generates a new credential value, (2) writes the new version to Vault KV v2, (3) updates `sops/postgres.sops.yaml` with the new encrypted value, (4) commits the updated SOPS file to git, (5) verifies the new credential reads back correctly
**And** the old credential version is retained in KV v2 version history (soft-deleted, not destroyed)
**And** the TTL of the new credential is ≤ 90 days (FR22)

**Given** `docs/runbooks/secret-rotation.md`,
**When** I read it,
**Then** the runbook covers both the Ansible-assisted procedure and a manual CLI fallback procedure (in case Ansible is unavailable)

---

### Story 4.5: Automated Rotation Hook Design [AR-S9 LOW, FR24 Growth]

As the operator,
I want the secret rotation infrastructure to include a documented hook point for future automated rotation (Growth phase),
So that when automated rotation is implemented in Growth it can be connected without rearchitecting the manual rotation procedure.

**Acceptance Criteria:**

**Given** `ansible/playbooks/day2-rotate-secrets.yml`,
**When** I inspect it,
**Then** there is a clearly marked `# GROWTH HOOK: automated rotation trigger point` comment at the location where automated scheduling would be wired in
**And** an inline comment documents: "Automated rotation in Growth phase should replace manual trigger with Vault Agent's `lease` stanza or an external scheduler calling this playbook"

**Given** `docs/runbooks/secret-rotation.md`,
**When** I read the Growth section,
**Then** it describes the two candidate automation mechanisms: Vault Agent `auto_auth` with lease renewal, and Ansible AWX/cron-triggered playbook scheduling
**And** the documentation notes which option is preferable given the operator constraint (Vault Agent preferred for application secrets; Ansible cron for operational rotation)

---

### Story 4.7: TLS Certificate Renewal Procedure

As the operator,
I want a documented and tested TLS certificate renewal procedure that replaces the self-signed CA and server certs when they approach expiry,
So that Vault TLS is renewed before expiry without requiring a full blast-and-repave.

**Acceptance Criteria:**

**Given** the self-signed CA and server certificate generated in Story 1.3,
**When** I follow the TLS certificate renewal runbook in `docs/runbooks/day0-bootstrap.md` (certificate section) or a dedicated `cert-renewal.md`,
**Then** the procedure covers: checking current certificate expiry (`openssl x509 -enddate`), running `tofu apply` targeting only `module.infra` to regenerate certs (triggering Vault listener restart), and verifying the new CA cert is re-distributed to all consumers

**Given** certificate replacement via `tofu apply -target module.infra`,
**When** the apply completes,
**Then** `curl --cacert new-ca.pem https://{vault_addr}/v1/sys/health` returns a valid response
**And** `vault status` with the updated `VAULT_CACERT` path returns `Sealed = false`
**And** all Ansible playbooks that reference `vault_ca_cert_path` are updated with the new CA cert path

**Given** the renewal procedure,
**When** I read the documentation,
**Then** it notes: the renewal schedule (before cert expiry date noted at bootstrap time), that consumers (OpenTofu, k3s, openclaw) that embed the old CA cert must re-trust the new CA, and that `tofu plan` will show changes only to the TLS cert resources (not the VM)

---

### Story 4.6: On-Demand Snapshot and Ansible Day-2 Runbooks (FR27, NFR14, NFR17)

As the operator,
I want an on-demand snapshot playbook and complete Day-2 operation runbooks,
So that all operational procedures are executable from git without prior context.

**Acceptance Criteria:**

**Given** `ansible/playbooks/day2-snapshot.yml`,
**When** I run it (`ansible-playbook day2-snapshot.yml`),
**Then** `vault operator raft snapshot save` is executed
**And** the snapshot is rsync'd to `{snapshot_host}` immediately
**And** a success/failure message is emitted with the snapshot filename and target path

**Given** all Day-2 runbooks in `docs/runbooks/`,
**When** I audit them against NFR14,
**Then** `day0-bootstrap.md`, `break-glass.md`, `blast-and-repave.md`, `snapshot-restore.md`, `secret-authoring.md`, `workload-onboarding.md`, `age-key-custody.md`, and `secret-rotation.md` all exist and are non-empty

---

## Epic 5: Disaster Recovery & Resilience

**Goal:** Operator can recover from any failure scenario — full VM destruction (blast-and-repave), Vault data corruption (snapshot restore), and authentication failure (break-glass) — with all procedures documented, exercised at least once, and producing a verified RTO measurement.

### Story 5.1: Blast-and-Repave Procedure [AR-S6 MEDIUM, FR13, FR30, NFR18]

As the operator,
I want a tested blast-and-repave procedure that reprovisioning Vault entirely from git via `tofu apply` after full VM destruction,
So that I can prove the system can be fully recovered in an emergency using only git and the age key.

**Acceptance Criteria:**

**Given** a running Vault instance with all three workload identities provisioned,
**When** I execute `tofu destroy && tofu apply` (simulating full VM destruction),
**Then** `tofu apply` completes with exit code 0 and the only human input required was setting `SOPS_AGE_KEY_FILE` (NFR18)
**And** after apply: `vault status` shows `Initialized = true` and `Sealed = false`
**And** all three AppRole identities are reprovisioned (verified by `vault auth list` and `vault policy list`)
**And** `ansible-playbook verify-isolation.yml` passes all three checks (D8 — FR11)

**Given** `docs/runbooks/blast-and-repave.md`,
**When** I read the "Verified Execution Record" section,
**Then** there is at least one completed execution log entry with: date, duration, outcome, and the RTO measurement for that run (NFR19)
**And** the runbook includes a "What can go wrong" section covering: SOPS age key unavailable, Proxmox API unreachable, Vault init output not captured

**Given** the blast-and-repave was exercised,
**When** I measure the wall-clock time from `tofu destroy` start to `verify-isolation.yml PASS`,
**Then** the time is recorded in the runbook and is used as the baseline RTO measurement

---

### Story 5.2: Snapshot Restore Procedure [FR29, NFR19, NFR13]

As the operator,
I want a tested snapshot restore procedure that recovers Vault data from a Raft snapshot without full VM reprovisioning,
So that I can recover from Vault data corruption without losing all stored secrets and without the full blast-and-repave time cost.

**Acceptance Criteria:**

**Given** a Raft snapshot taken via Story 4.3 (stored on `{snapshot_host}`),
**When** I follow the snapshot restore procedure from `docs/runbooks/snapshot-restore.md`,
**Then** `vault operator raft snapshot restore <snapshot_file>` completes successfully
**And** the three-part acceptance test passes: (1) secrets readable (`vault kv get` returns known values), (2) AppRole auth functional (login succeeds for all three workloads), (3) audit log active (`vault audit list` shows backend, new operations are logged — FR19)

**Given** the snapshot restore was exercised,
**When** I measure the wall-clock time from initiating restore to passing the three-part acceptance test,
**Then** the time is ≤ 4 hours (NFR13)
**And** the measured RTO is recorded in `docs/runbooks/snapshot-restore.md` execution log

**Given** `docs/runbooks/snapshot-restore.md`,
**When** I read it,
**Then** it covers: when to use snapshot restore vs blast-and-repave (decision guide), prerequisites, the restore command, the three-part acceptance test procedure, and how to handle a failed restore

---

### Story 5.3: Break-Glass Procedure [AR-S5 MEDIUM, FR31, FR32, NFR14]

As the operator,
I want a documented and exercised break-glass procedure for unsealing Vault and rotating the root token when normal authentication paths are unavailable,
So that I can recover operational access in an emergency without leaving persistent root credentials exposed.

**Acceptance Criteria:**

**Given** `docs/runbooks/break-glass.md`,
**When** I follow the procedure during a simulated break-glass event (e.g., all AppRole tokens expired),
**Then** the procedure covers: how to retrieve the unseal key from offline custody, `vault operator unseal` to unseal, generating and using a root token for the duration of the session, the mandatory `ops-log.md` entry (what, why, when), and root token rotation/revocation at session end (FR32)
**And** `vault token revoke <root_token>` is the final command in the procedure with a confirmation step

**Given** the break-glass procedure was exercised at least once,
**When** I inspect `docs/ops-log.md`,
**Then** there is at least one completed entry with: ISO 8601 timestamp, reason for break-glass use, operations performed, and revocation confirmation

**Given** `docs/ops-log.md`,
**When** Vault audit backend is active (FR20),
**Then** every break-glass session is cross-referenced between `ops-log.md` (operator record) and Vault audit log (system record)

---

### Story 5.4: Compliance Deviation Records (FR21, NFR14)

As the operator,
I want formal deviation documentation for the separation of duties (SoD) deviation and the Shamir single-key deviation,
So that both deviations are formally acknowledged with documented compensating controls — meeting the financial-sector-bar standard for deviation management.

**Acceptance Criteria:**

**Given** `docs/deviations/sod-deviation.md`,
**When** I read it,
**Then** it documents: the deviation (single operator, no separation of duties), the standard it deviates from (NIST principle of least privilege / regulatory SoD requirement), the rationale (homelab single operator), and the compensating controls (all privileged ops policy-driven, logged, code-reviewed automation only, no ad-hoc root access — cross-references break-glass ops-log requirement)

**Given** `docs/deviations/dual-control-deviation.md`,
**When** I read it,
**Then** it documents: the Shamir Secret Sharing deviation (1-of-1 threshold), the NIST SP 800-57 dual-control standard, the rationale, and the compensating controls (unseal key in offline custody with documented custody runbook)

---

### Story 5.5: Dependency Version Management (FR37, FR38, NFR21, NFR22, NFR23)

As the operator,
I want pinned versions for all security-critical components and a version exception documentation pattern,
So that the version state of every component is auditable from git and any deviation from latest is formally acknowledged.

**Acceptance Criteria:**

**Given** `.terraform.lock.hcl`,
**When** I inspect it,
**Then** `hashicorp/vault`, `hashicorp/proxmox`, and `hashicorp/tls` providers all have pinned version constraints (no `>= x.x` floating constraints)
**And** the Vault binary version is pinned in `variables.tf` default: `"1.21.4"` with a comment: `# Pinned per architecture baseline 2026-03-24; upgrade via tofu plan → review → tofu apply`

**Given** `docs/ops-log.md`,
**When** a version exception is declared (component pinned below latest),
**Then** the exception entry includes: component name, pinned version, current latest, reason for exception, and target resolution version (NFR23)

**Given** `variables.tf`,
**When** I read the `vault_version` variable,
**Then** the upgrade procedure comment cross-references: `# Upgrade procedure: update default value → tofu plan → review for schema changes → tofu apply` (NFR22)

---

## Implementation Readiness Checks

All 39 FRs are covered across the 5 epics above.

| FR | Epic | Story |
|---|---|---|
| FR1 | 2 | 2.3 |
| FR2 | 2 | 2.4 |
| FR3 | 2/3 | 2.4, 3.3 |
| FR4 | 2 | 2.3 |
| FR5 | 2 | 2.3, 2.5 |
| FR6 | 3 | 3.3, 3.4, 3.5 |
| FR7 | 3 | 3.3, 3.4, 3.5 |
| FR8 | 3 | 3.6 |
| FR9 | 3 | 3.6 |
| FR10 | 3 | 3.1 |
| FR11 | 3 | 3.7 |
| FR12 | 1 | 1.8 |
| FR13 | 5 | 5.1 |
| FR14 | 1 | 1.6, 1.7, 1.8 |
| FR15 | 4 | 4.1–4.6 |
| FR16 | 3 | 3.3 |
| FR17 | 3 | 3.1 |
| FR18 | 3 | 3.1 |
| FR19 | 5 | 5.2 |
| FR20 | 3 | 3.1, 5.3 |
| FR21 | 5 | 5.4 |
| FR22 | 3 | 3.2, 3.3 |
| FR23 | 4 | 4.4 |
| FR24 | 4 | 4.5 (hook) |
| FR25 | 2 | 2.6 |
| FR26 | 2 | 2.5 |
| FR27 | 4 | 4.6 |
| FR28 | 4 | 4.3 |
| FR29 | 5 | 5.2 |
| FR30 | 5 | 5.1 |
| FR31 | 5 | 5.3 |
| FR32 | 5 | 5.3 |
| FR33 | 1 | 1.5 |
| FR34 | 1 | 1.6 |
| FR35 | 4 | 4.2 |
| FR36 | 4 | 4.3 (stub) |
| FR37 | 5 | 5.5 |
| FR38 | 5 | 5.5 |
| FR39 | — | Deferred to `terminus-infra-dependency-health` |

**FR coverage: 38/39 (FR39 intentionally deferred per architecture)**

**NFR coverage:** All 23 NFRs are covered across stories. Security NFRs (1–6) covered in Epics 1–2. Reliability NFRs (7–10) covered in Epics 1, 4, 5. Performance NFRs (11–13) covered in Epics 1, 5. Operability NFRs (14–17) covered by runbook stories in each epic. Recoverability NFRs (18–20) covered in Epic 5. Maintainability NFRs (21–23) covered in Story 5.5.
