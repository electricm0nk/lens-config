---
stepsCompleted: [step-01-init, step-02-context, step-03-starter, step-04-decisions, step-05-patterns, step-06-structure, step-07-validation]
inputDocuments:
  - docs/terminus/infra/secrets/prd.md
workflowType: 'architecture'
project_name: 'terminus-infra-secrets'
user_name: 'Todd'
date: '2026-03-23'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements (39 FRs across 8 areas):**

| Area | FRs | Architectural Weight |
|---|---|---|
| Secret Storage & Delivery | FR1–FR5 | KV v2 path hierarchy, SOPS authoring pipeline |
| Identity & Access Control | FR6–FR11 | AppRole lifecycle, policy-as-code, default-deny enforcement |
| Provisioning & Automation | FR12–FR16 | OpenTofu as sole management interface; bootstrap idempotency |
| Audit & Compliance | FR17–FR21 | Structured JSON audit log, 100% operation coverage, deviation records |
| Secret Lifecycle Management | FR22–FR26 | TTL enforcement, manual/automated rotation, SOPS destruction procedure |
| Disaster Recovery & Resilience | FR27–FR32 | Two distinct DR paths (blast-and-repave vs snapshot restore), break-glass |
| Observability & Health | FR33–FR36 | `/v1/sys/health`, Consul health check readiness, SIEM-compatible log format |
| Dependency & Patch Management | FR37–FR39 | Version pinning in lock files, CVE query (deferred to future initiative) |

**Non-Functional Requirements driving architecture directly:**

- **NFR1/NFR2**: AES-256-GCM at rest (Raft), TLS required on Vault listener — no plaintext HTTP
- **NFR4**: Root token must not persist — architectural constraint on bootstrap and break-glass flows
- **NFR10**: Raft snapshots must not be correlated with VM loss — off-VM snapshot storage required
- **NFR15**: No manual Vault configuration outside break-glass — OpenTofu is the sole management interface
- **NFR18**: Blast-and-repave = single `tofu apply` — the only human input permitted is providing the age key
- **NFR21/NFR22**: All component versions pinned; upgrades governed by `tofu plan` → review → `tofu apply` cycle

**Scale & Complexity:**

- **Complexity level:** High — foundational, cross-cutting, all downstream initiatives blocked on this capability
- **Primary domain:** Infrastructure Platform / Operations Security
- **Delivery phases:** 3 (MVP → Growth → Vision); MVP is fully load-bearing before Growth can begin
- **Estimated architectural components:** 12+ discrete components
- **DR testability is an MVP design constraint, not a scope item.** Both blast-and-repave and snapshot restore execution paths must be architecturally designed and exercisable in MVP. Growth exit criteria that depend on "MVP DR-tested" are blocked if the DR paths are bolted on rather than designed in.

### Technical Constraints & Dependencies

**Non-negotiable from PRD:**
- Vault OSS standalone single-node — not HA; OSS HA does not transfer to Enterprise skills
- Raft Integrated Storage as storage backend — Consul explicitly NOT Vault storage
- AppRole only (MVP auth method) — no userpass, LDAP, SSO
- KV v2 — versioning, soft-delete, metadata required
- Vault UI: out of scope (attack surface)
- Vault Enterprise features: out of scope (namespaces, auto-unseal HSM, DR replication)
- Path hierarchy `secret/terminus/{domain}/{service}/{key}` is a stable, breaking-change-sensitive contract
- `VAULT_ADDR` / `VAULT_TOKEN` must never appear in workload environment variables — Vault Agent injection only

**Bootstrap constraint (age key):** The age private key must be available in the `tofu apply` execution environment via a mechanism that does not require Vault to already exist (e.g., operator provides via env var, file mount, or SOPS-decrypted local file). This is a hard architectural constraint — no circular dependency path is acceptable.

**Audit log retention:** Architecture must specify on-disk log rotation parameters (max file size, retention window, rotation schedule) as part of VM provisioning design. PRD mandates minimum 90-day local retention; archival policy is TBD but the log rotation strategy is required before Day-0 — unconfigured log growth is an ops surprise.

**Audit log schema stability:** Vault audit log JSON schema version must be documented at architecture time. Vault version upgrades may introduce schema drift — the architecture document must note the schema baseline and flag upgrade notes as a maintenance obligation.

**Toolchain (versions pinned):**
- Vault 1.21.4 | OpenTofu 1.11.0 | SOPS + age | Consul 1.22.5 | Ansible-core 2.19

**Platform:** Proxmox VMs | Target repo: `terminus.infra`

### Cross-Cutting Concerns Identified

| Concern | Affects |
|---|---|
| **Bootstrap problem** — age key is the trust anchor before Vault exists; delivery mechanism must not require Vault | Day-0 provisioning, SOPS pipeline, break-glass, blast-and-repave |
| **TLS everywhere** | Vault listener, all API clients, Consul health checks |
| **Audit logging on every operation** | All 8 capability areas; log must persist through restart/restore; schema pinned at architecture time |
| **AppRole as universal auth pattern** | All consumers — OpenTofu, k3s, OpenClaw, future workloads |
| **Policy-as-code / default-deny** | All AppRole identities; every policy committed to git before activation |
| **Two-path DR** | Blast-and-repave (git+SOPS source) and snapshot restore (Raft data source) — separate procedures, separate tests, both designed in MVP |
| **Monitoring-readiness** | Log format, health endpoint, Consul integration — correct in MVP even though monitoring is deferred |
| **Path stability contract** | `secret/terminus/{domain}/{service}/{key}` — path renames are orchestrated migrations, not renames |
| **Isolation verification suite** | First-class deliverable — architectural home must be decided (Ansible playbook, standalone script, or pipeline check); suite runs on every reprovision as part of three-part acceptance test |

---

## Starter Template Evaluation

### Primary Technology Domain

Infrastructure Platform (IaC) — no traditional application starter template applies.
Primary composition surfaces: OpenTofu modules, Ansible playbooks, HCL policy files.

### Starter Approach

This project does not use a scaffolded application starter (Next.js, NestJS, etc.).
The "starter" decision is the OpenTofu module structure and the Vault provider
bootstrapping strategy — these establish the architectural skeleton that all
other decisions build on.

### Key Evaluated Options

| Option | Description | Trade-off |
|---|---|---|
| Monolithic Vault module | Single OpenTofu module: VM + Vault init + policies + AppRoles | Simple targeting; blast-and-repave destroys everything together |
| Layered modules | Separate modules: `infra/` (VM, networking), `vault-core/` (init, auth, mounts), `vault-config/` (policies, AppRoles, secrets) | Surgical reprovisioning; config layer reapplied without VM reprovision |
| Community module (`hashicorp/vault` provider examples) | HashiCorp reference patterns for Vault on VMs | Good provider usage patterns; no Proxmox-specific scaffolding |
| Ansible Galaxy (`geerlingguy.vault`) | Community role for Vault installation and config | Covers OS-level install; less control over init/unseal sequence |

### Selected Approach: Layered OpenTofu Modules

**Rationale:** The blast-and-repave requirement (FR13, NFR18) demands that the
config layer (policies, AppRoles, secret paths) be independently targetable from
the infrastructure layer (VM, networking). A monolithic module cannot satisfy
`tofu destroy -target vault_config && tofu apply` without also destroying the VM.
Layered modules make the DR paths explicit in code.

**Module structure (to be refined in architectural decisions step):**

```
modules/
  infra/          # Proxmox VM, networking, TLS cert
  vault-core/     # Vault init, unseal, auth method enablement, KV v2 mount
  vault-config/   # Policies (HCL), AppRoles, secret seeding from SOPS
ansible/
  playbooks/      # Day-2 ops: rotation, cert renewal, snapshot scheduling
  roles/          # Custom roles — no Galaxy dependency for security-critical ops
```

**Vault provider bootstrapping strategy:** Deferred to architectural decisions
step — this is the bootstrap problem resolution and merits dedicated analysis.

---

## Core Architectural Decisions

### Decision Priority Analysis

**Already Decided (from PRD + step 3):**
- Vault OSS standalone single-node; Raft Integrated Storage; AppRole (MVP); KV v2; no Vault UI
- Layered OpenTofu modules (`infra/`, `vault-core/`, `vault-config/`); custom Ansible roles
- Toolchain versions pinned: Vault 1.21.4, OpenTofu 1.11.0, Consul 1.22.5, Ansible-core 2.19

**Critical Decisions (block implementation):**
- D1: Age key delivery, D2: Vault init strategy, D4: State backend, D6: TLS strategy

**Important Decisions (shape Day-2 operations):**
- D3: Unseal key strategy, D5: Snapshot storage, D7: Audit log rotation, D8: Verification suite

**Deferred:**
- Auto-unseal (no Cloud KMS; Transit requires second Vault); Step-CA PKI; CVE automation (`terminus-infra-dependency-health`)

### Bootstrap & Credential Delivery

**D1 — Age key delivery into `tofu apply`**
- **Decision:** `SOPS_AGE_KEY_FILE` environment variable, set by operator before running `tofu apply`
- **Rationale:** Explicit, auditable, keeps age key on operator workstation only; no wrapper script complexity. Age key custody location (offline storage) is documented separately in the break-glass/custody runbook.
- **Constraint satisfied:** No circular dependency; age key availability does not require Vault to exist.

**D2 — Vault initialization strategy**
- **Decision:** OpenTofu `null_resource` with `remote-exec` provisioner in `vault-core/` module runs `vault operator init`; init output (unseal keys, root token) emitted to `tofu apply` stdout for operator capture
- **Rationale:** Preserves single `tofu apply` → operational Vault (FR12, NFR18). Root token is passed directly into the Vault provider configuration for subsequent resource creation, then revoked as the final resource in `vault-core/`.
- **Operator obligation:** Capture unseal keys from `tofu apply` stdout; store per custody documentation.

**D3 — Unseal key strategy**
- **Decision:** Shamir Secret Sharing, 1 key share, 1 threshold (single operator)
- **Rationale:** Single operator; custody ceremony documented; additional shares add no security benefit without multiple custodians. Documented as known deviation from NIST SP 800-57 dual-control (accepted in PRD risk register).
- **Break-glass:** Unseal key stored offline per custody runbook; `vault operator unseal` is the break-glass procedure.

### State & Storage

**D4 — OpenTofu state backend**
- **Decision:** Local state file; SOPS-encrypted before commit to git (state contains AppRole IDs, Vault tokens)
- **Rationale:** State contains sensitive values and must be encrypted. SOPS+age pipeline handles this consistently. Git provides backup and version history. No additional infrastructure required.
- **Implementation note:** `.terraform/` excluded from git; only `terraform.tfstate` committed (SOPS-encrypted). `make encrypt-state` target in Makefile documents the encrypt-before-commit discipline.

**D5 — Snapshot storage location**
- **Decision:** Ansible cron playbook runs `vault operator raft snapshot save` and rsync to a separate Proxmox host (not the Vault VM)
- **Rationale:** Satisfies NFR10 (snapshot and VM loss must not be correlated). Ansible playbook = documented, reproducible, testable procedure. Target Proxmox host is a deployment-time configuration variable.
- **Retention:** 7 daily snapshots retained; oldest pruned by cron. 90-day archive snapshot taken monthly.

### TLS & Certificates

**D6 — TLS certificate strategy**
- **Decision:** OpenTofu `tls` provider generates self-signed CA and server cert; CA private key protected via SOPS-encrypted state (D4); CA public cert committed to git unencrypted for consumer trust distribution
- **Rationale:** Fully IaC-driven; cert rolls on reprovision with no manual steps. Self-signed CA acceptable for homelab. CA cert in git allows consumers (OpenTofu, Ansible, k3s) to trust Vault TLS without manual cert distribution.
- **Upgrade path:** When `terminus-infra-monitoring` introduces Step-CA or external PKI, Vault provider config updated to reference new certs — no rearchitecting required.

### Operational Infrastructure

**D7 — Audit log rotation**
- **Decision:** `logrotate` configured via Ansible; daily rotation; gzip compression; 90-day retention (rotate 90); 100MB forced-rotation threshold
- **Rationale:** Standard, simple, no external dependency. Configured as code (Ansible role). Satisfies PRD 90-day minimum local retention. Future log-shipping to `terminus-infra-monitoring` adds promtail/Filebeat without replacing logrotate.
- **Audit log schema:** Vault 1.21.4 JSON audit log schema documented as baseline. Schema version noted in architecture; upgrade notes required for Vault version bumps.

**D8 — Isolation verification suite**
- **Decision:** Ansible playbook `ansible/playbooks/verify-isolation.yml`; invoked as final step of blast-and-repave and snapshot restore procedures; also runnable ad-hoc
- **Rationale:** Consistent with operational model (all Day-2 ops are Ansible). Output to stdout provides audit trail. Playbook tests: (1) OpenTofu AppRole reads its scoped paths, (2) OpenClaw AppRole denied outside its namespace, (3) audit log contains both permitted and denied entries.
- **Scope:** First-class MVP deliverable.

### Decision Impact Analysis

**Implementation sequence driven by decisions:**
1. `infra/` module → Proxmox VM + network + TLS cert (D6)
2. `vault-core/` module → Vault init via null_resource (D2), unseal (D3), auth method enablement, KV v2 mount
3. `vault-config/` module → Policies, AppRoles, secret seeding from SOPS (D1 prerequisite: age key in env)
4. State encrypted and committed after successful apply (D4 discipline)
5. Snapshot cron configured via Ansible post-provision (D5)
6. Audit log rotation configured via Ansible post-provision (D7)
7. Isolation verification suite runs as final provision step (D8)

**Cross-component dependencies:**
- D1 (age key) is a prerequisite for D2 (init), D4 (state encrypt), and all SOPS operations
- D2 (init output) feeds root token into Vault provider — root token lifecycle and `vault-core/` resource ordering must be explicit
- D6 (TLS cert) must be provisioned in `infra/` before `vault-core/` can configure Vault listener — module dependency ordering required in root module
- D8 (verification suite) depends on D2, D3, and `vault-config/` being complete — always last in the run sequence

---

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified

10 areas where implementation agents could make inconsistent choices without
explicit rules. All patterns below are binding for all implementation work.

### Naming Patterns

**Vault Path Hierarchy — INVARIANT**
```
secret/terminus/{domain}/{service}/{key}
```
- `domain`: infra | platform | agent
- `service`: postgres | k3s | openclaw | opentofu | (new services use lowercase-hyphenated)
- `key`: the specific credential name (e.g., `app-role`, `api-key`, `tls-cert`)
- Path renames are breaking changes — require coordinated migration, never silent rename

**Vault AppRole Naming**
- Pattern: `{service}` (lowercase, hyphenated, no prefix/suffix)
- Example: `opentofu`, `k3s`, `openclaw`
- Policy file name matches: `{service}.hcl`
- OpenTofu resource name matches: `vault_approle_auth_backend_role.{service}`

**OpenTofu Resource Naming**
- Pattern: `{resource_type}.{service}` — no adjectives or suffixes unless disambiguating
- Examples: `vault_policy.opentofu`, `vault_kv_secret_v2.postgres_app_role`, `vault_approle_auth_backend_role.k3s`
- Module output names: `vault_addr`, `vault_ca_cert_pem`, `approle_role_id_{service}`

**OpenTofu Variable Naming**
- All lowercase snake_case: `vault_addr`, `vault_token`, `proxmox_node`, `sops_age_key_file`
- Injected from environment: variable name matches the env var name minus the `TF_VAR_` prefix
- No abbreviation unless the abbreviation is the canonical name (e.g., `addr` not `address`)

**HCL Policy Files**
- Filename: `{service}.hcl` — matches AppRole name exactly
- Location: `vault-config/policies/{service}.hcl`
- Header comment in every file: `# Policy for {service} AppRole — read-only on {paths}`

**SOPS File Naming**
- Pattern: `{subsystem}.sops.yaml` — `.sops.yaml` suffix is mandatory (not `.enc.yaml`)
- Location: `sops/{subsystem}.sops.yaml` at repo root
- Example: `sops/postgres.sops.yaml`, `sops/bootstrapcreds.sops.yaml`

**Ansible Variables**
- All lowercase snake_case: `vault_version`, `vault_addr`, `snapshot_target_host`
- Role defaults in `roles/{role}/defaults/main.yml`; no inline defaults in playbooks
- No uppercase variable names except `VAULT_TOKEN` / `VAULT_ADDR` when passed as shell env

**Ansible Task Naming**
- Imperative verb phrase, title case: `Install Vault Binary`, `Configure Audit Log`, `Run Isolation Verification`
- Play names: `Vault {Layer} — {Action}` (e.g., `Vault Core — Initialize and Unseal`)

### Structure Patterns

**OpenTofu Module Layout**
```
terminus.infra/
  modules/
    infra/              # Proxmox VM, networking, TLS cert generation
      main.tf
      variables.tf
      outputs.tf
    vault-core/         # Vault init, unseal, auth method enablement, KV v2 mount
      main.tf
      variables.tf
      outputs.tf
    vault-config/       # Policies, AppRoles, secret seeding
      main.tf
      variables.tf
      outputs.tf
      policies/         # {service}.hcl files
  main.tf               # Root module — wires modules together, declares providers
  variables.tf          # All input variables (fed from env / CI)
  outputs.tf            # Surfaces vault_addr, ca_cert_pem for consumers
  terraform.tfstate     # SOPS-encrypted before commit
  .terraform.lock.hcl   # Committed unencrypted (no secrets)
```

**Ansible Layout**
```
terminus.infra/
  ansible/
    playbooks/
      day0-provision.yml        # Called after tofu apply — logrotate, snapshot cron
      day2-rotate-secrets.yml   # Manual rotation procedure
      day2-snapshot.yml         # On-demand snapshot (also called by cron)
      verify-isolation.yml      # Three-part acceptance test
    roles/
      vault-logrotate/          # Configures audit log rotation
      vault-snapshot/           # Configures snapshot cron + rsync
    inventory/
      hosts.yml                 # Vault VM address; populated from tofu outputs
```

**SOPS Layout**
```
terminus.infra/
  sops/
    .sops.yaml              # SOPS config: recipient age pubkey, path rules
    bootstrapcreds.sops.yaml
    postgres.sops.yaml
    (per-subsystem files)
```

### Format Patterns

**OpenTofu Outputs — Public Contract**

All outputs that downstream consumers depend on must be documented in `outputs.tf`
with a `description` field. No silent outputs. Outputs are the API of the module.

Required root module outputs:
- `vault_addr` — `https://{vm_ip}:8200`
- `vault_ca_cert_pem` — PEM string; consumers write to trusted CA store
- `approle_role_id_{service}` — one output per registered AppRole

**Git Commit Messages**
- Pattern: `{type}({scope}): {description}`
- Types: `feat`, `fix`, `docs`, `config`, `refactor`, `test`, `chore`
- Scope for this repo: `vault-core`, `vault-config`, `infra`, `ansible`, `sops`, `docs`
- Examples: `feat(vault-config): add openclaw approle and isolation policy`
           `docs(ansible): add snapshot restore runbook`

**SOPS File Format**
- All SOPS files are YAML (not JSON)
- Top-level keys match Vault path leaf names: `app_role_password`, `api_key`
- Plaintext structure documented in comment at top of encrypted file (`# Keys: app_role_password, api_key`)

### Process Patterns

**AppRole Onboarding Checklist** (every new workload follows this sequence)
1. Create `vault-config/policies/{service}.hcl` — define exact paths and capabilities
2. Add `vault_approle_auth_backend_role.{service}` resource in `vault-config/main.tf`
3. Add `approle_role_id_{service}` output in `vault-config/outputs.tf`
4. Add isolation test case to `ansible/playbooks/verify-isolation.yml`
5. Run `tofu plan` → review → `tofu apply`
6. Run `ansible-playbook verify-isolation.yml` — must pass before workload goes live

**State Encrypt-Before-Commit Rule**

`terraform.tfstate` must be SOPS-encrypted before every `git add`. The workflow:
```
tofu apply
sops --encrypt terraform.tfstate > terraform.tfstate.tmp && mv terraform.tfstate.tmp terraform.tfstate
git add terraform.tfstate
```
A `Makefile` target `make encrypt-state` encodes this. Never commit unencrypted state.

**Break-Glass Token Lifecycle Rule**

Any session that uses the root token or a break-glass token must end with:
1. `vault token revoke <token>` or root token rotation
2. A timestamped entry in `ops-log.md` recording: what, why, when, revocation confirmed

This is not optional — it is the compensating control for the SoD deviation.

---

## Project Structure & Boundaries

### Complete Project Directory Structure

```
terminus.infra/
├── README.md                          # Project overview + quickstart
├── Makefile                           # encrypt-state, plan, apply, verify targets
├── .gitignore                         # Excludes: .terraform/, *.tfstate.bak, plaintext key files
├── .terraform.lock.hcl                # Pinned provider versions — committed unencrypted
├── main.tf                            # Root module — wires infra/ + vault-core/ + vault-config/
├── variables.tf                       # All input vars: proxmox_node, vault_version, snapshot_host, etc.
├── outputs.tf                         # vault_addr, vault_ca_cert_pem, approle_role_id_{service}
├── terraform.tfstate                  # SOPS-encrypted before every commit (make encrypt-state)
│
├── modules/
│   ├── infra/                         # FR12: VM + networking + TLS cert
│   │   ├── main.tf                    # proxmox_vm_qemu, tls_self_signed_cert, tls_private_key
│   │   ├── variables.tf
│   │   └── outputs.tf                 # vm_ip, ca_cert_pem, ca_private_key_pem (sensitive)
│   │
│   ├── vault-core/                    # FR12, FR13: Vault init/unseal + auth + mounts
│   │   ├── main.tf                    # null_resource(vault-init), vault_auth_backend(approle),
│   │   │                              #   vault_mount(secret-kv2), null_resource(revoke-root-token)
│   │   ├── variables.tf               # vault_addr, vault_ca_cert, unseal_key_shares=1
│   │   └── outputs.tf                 # vault_initialized (bool)
│   │
│   └── vault-config/                  # FR6–FR11, FR2, FR17: Policies + AppRoles + audit
│       ├── main.tf                    # vault_policy.*, vault_approle_auth_backend_role.*,
│       │                              #   vault_audit (file backend)
│       ├── variables.tf               # vault_addr, vault_token (root — short-lived), sops_secrets_*
│       ├── outputs.tf                 # approle_role_id_{service} for each registered workload
│       └── policies/
│           ├── opentofu.hcl           # FR7: read secret/terminus/infra/* + sys/leases/renew
│           ├── k3s.hcl                # FR7: read secret/terminus/infra/k3s/*
│           └── openclaw.hcl           # FR7: read secret/terminus/agent/openclaw/* — deny all else
│
├── sops/
│   ├── .sops.yaml                     # age recipient pubkey + path glob rules
│   ├── bootstrapcreds.sops.yaml       # D1: bootstrap material (unseal key custody backup if needed)
│   └── postgres.sops.yaml             # FR1: example subsystem — app_role_password, etc.
│
├── ansible/
│   ├── inventory/
│   │   └── hosts.yml                  # vault_vm host + vars (vault_addr from tofu output)
│   │
│   ├── playbooks/
│   │   ├── day0-provision.yml         # FR28, D7: post-tofu-apply — logrotate + snapshot cron
│   │   ├── day2-rotate-secrets.yml    # FR23: manual rotation procedure
│   │   ├── day2-snapshot.yml          # FR27: on-demand snapshot + rsync to snapshot_host
│   │   └── verify-isolation.yml       # D8: three-part acceptance test (FR11, FR29, FR30)
│   │
│   └── roles/
│       ├── vault-logrotate/           # D7: logrotate config — 90-day / 100MB / daily / gzip
│       │   ├── defaults/main.yml
│       │   ├── tasks/main.yml
│       │   └── templates/vault-audit.logrotate.j2
│       └── vault-snapshot/            # D5: snapshot cron + rsync to snapshot_host
│           ├── defaults/main.yml      #   snapshot_host, snapshot_retention_days=7
│           ├── tasks/main.yml
│           └── templates/vault-snapshot.cron.j2
│
└── docs/
    ├── ops-log.md                     # FR20: running log of all privileged operations
    ├── runbooks/
    │   ├── day0-bootstrap.md          # Journey 1: complete bootstrap procedure
    │   ├── break-glass.md             # Journey 4: unseal + root token rotation + ops-log entry
    │   ├── blast-and-repave.md        # Journey 5a: full reprovisioning from git
    │   ├── snapshot-restore.md        # Journey 5b: raft snapshot restore + three-part test
    │   ├── secret-authoring.md        # Journey 2: SOPS → git → Vault sync workflow
    │   ├── workload-onboarding.md     # Journey 3: AppRole checklist from step 5
    │   ├── age-key-custody.md         # NFR6, D3: key location, rotation procedure, recovery
    │   └── secret-rotation.md         # FR23, FR25: rotation procedure (manual; automated in Growth)
    └── deviations/
        ├── sod-deviation.md           # FR21: SoD deviation + compensating controls record
        └── dual-control-deviation.md  # FR21: NIST dual-control deviation + compensating controls
```

### Architectural Boundaries

**Module Dependency Chain**
```
root main.tf
  └── module "infra"        → outputs: vm_ip, ca_cert_pem
        ↓ (depends_on)
  └── module "vault-core"   → inputs: vm_ip, ca_cert_pem
        ↓ (depends_on)       → outputs: vault_initialized
  └── module "vault-config" → inputs: vault_addr, vault_token (root, short-lived)
```
Modules must never be applied out of order. Root `main.tf` enforces ordering via
`depends_on`. Each module is independently destroyable from the root:
- `tofu destroy -target module.vault-config` → wipes policies/AppRoles; VM intact
- `tofu destroy -target module.vault-core` → wipes Vault state; VM intact
- `tofu destroy -target module.infra` → destroys entire VM (full blast-and-repave)

**Provider Boundaries**
- `hashicorp/proxmox` — used only in `modules/infra/`
- `hashicorp/vault` — used only in `modules/vault-core/` and `modules/vault-config/`
- `hashicorp/tls` — used only in `modules/infra/`
- No provider used in more than one module layer (no cross-layer provider bleed)

**Secrets Boundary**
- Sensitive values (CA private key, root token, AppRole secret-ids) flow as OpenTofu
  `sensitive = true` outputs — never written to non-sensitive outputs or log files
- SOPS secrets decrypted transiently at `tofu apply` time — never persisted in plaintext on disk
- All `sensitive` outputs marked as such in `outputs.tf` with `sensitive = true`

### Requirements to Structure Mapping

| FR Area | Primary Location | Secondary Location |
|---|---|---|
| FR1–FR5 (Secret Storage) | `modules/vault-config/`, `sops/` | `docs/runbooks/secret-authoring.md` |
| FR6–FR11 (Identity & Access) | `modules/vault-config/policies/`, `modules/vault-config/main.tf` | `docs/runbooks/workload-onboarding.md` |
| FR12–FR16 (Provisioning) | `modules/infra/`, `modules/vault-core/`, `modules/vault-config/` | `docs/runbooks/day0-bootstrap.md` |
| FR17–FR21 (Audit & Compliance) | `modules/vault-config/main.tf` (audit backend) | `docs/deviations/`, `docs/ops-log.md` |
| FR22–FR26 (Secret Lifecycle) | `ansible/playbooks/day2-rotate-secrets.yml` | `docs/runbooks/secret-rotation.md` |
| FR27–FR32 (DR & Resilience) | `ansible/playbooks/day2-snapshot.yml`, `ansible/roles/vault-snapshot/` | `docs/runbooks/blast-and-repave.md`, `docs/runbooks/snapshot-restore.md` |
| FR33–FR36 (Observability) | `modules/vault-core/` (health config), `ansible/roles/vault-logrotate/` | Wired to `terminus-infra-monitoring` in Growth |
| FR37–FR39 (Dependency Mgmt) | `.terraform.lock.hcl` | `docs/ops-log.md` (exceptions log) |

### Integration Points

**Downstream consumer integration (public contract):**
- Consumers read `vault_addr` and `vault_ca_cert_pem` from root `outputs.tf`
- Consumers authenticate via their AppRole (role-id from `approle_role_id_{service}` output)
- Path contract: `secret/terminus/{domain}/{service}/{key}` — stable, documented in README

**External dependencies:**
- Proxmox API (VM provisioning) — endpoint configured in `variables.tf`
- age key (operator-provided via `SOPS_AGE_KEY_FILE` before `tofu apply`)
- Snapshot target host (separate Proxmox host, not the Vault VM) — `snapshot_host` variable

**Monitoring integration points (deferred, no rearchitecting needed):**
- Audit log: `/var/log/vault/audit.log` on Vault VM — promtail/Filebeat connects here
- Health: `https://{vault_addr}/v1/sys/health` — Consul + future Prometheus connect here
- Consul health check: `vault_service_check` resource added to `vault-core` in Growth

---

## Architecture Validation Results

### Coherence Validation ✅

All eight architectural decisions (D1–D8) are mutually compatible. The module
dependency chain (`infra` → `vault-core` → `vault-config`) is acyclic and
enforced via `depends_on` in the root module. Bootstrap constraint (D1) cleanly
breaks the circular dependency. Root token lifecycle (D2) is explicitly scoped
within `vault-core/` — initialized, used for configuration, revoked as the final
resource in the module.

**Implementation note — Vault provider alias required:** During `vault-core/`
execution, the Vault provider must be configured with the root token captured
from `vault operator init`. The implementation must declare a provider alias
(e.g., `provider "vault" { alias = "root" }`) in the root module, seeded with
the init output, and pass it as a `providers` argument to `module "vault-core"`.
This prevents the unreachable-provider error that occurs if the provider is
configured before Vault is initialized.

### Requirements Coverage Validation ✅

All 39 FRs and 23 NFRs are architecturally covered. Deferred items are
explicitly scoped:
- FR24 (automated rotation) → Growth phase; manual rotation procedure in MVP
- FR36 (Consul health check) → Growth phase; health endpoint designed in MVP
- FR39 (CVE automation) → `terminus-infra-dependency-health` future initiative
- NFR9 (workload graceful degradation) → Consumer-side responsibility; documented
  in README as consumer integration obligation: *"Workloads must cache last-known
  credentials and degrade gracefully when `vault_addr` is unreachable."*

### Implementation Readiness Validation ✅

**Decision completeness:** All 8 critical decisions (D1–D8) documented with
rationale, constraints satisfied, and implementation notes.

**Pattern completeness:** 10 naming conflict points resolved; AppRole onboarding
checklist, state-encrypt-before-commit, and break-glass token lifecycle rules
eliminate the three most likely process deviations.

**Structure completeness:** Complete directory tree with 30+ named files, all
carrying FR references. FR-to-directory mapping table explicit.

### Gap Analysis Results

| Gap | Priority | Resolution |
|---|---|---|
| Vault provider alias pattern for root token handoff | High | Implementation note added to coherence section — Amelia must follow |
| NFR9 consumer graceful degradation | Medium | Document in README as consumer integration obligation |
| Audit log JSON schema baseline | Low | Vault 1.21.4 schema is the baseline per D7; schema version noted |
| Snapshot retention for monthly archives | Low | D5 specifies 7 daily + monthly 90-day archive; Ansible role `defaults/main.yml` encodes this |

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] 39 FRs and 23 NFRs analyzed for architectural implications
- [x] Scale and complexity assessed (High / Infrastructure Platform)
- [x] Technical constraints identified (10 non-negotiable PRD constraints)
- [x] Cross-cutting concerns mapped (9 concerns — bootstrap, TLS, audit, AppRole, policy-as-code, two-path DR, monitoring-readiness, path stability, verification suite)

**✅ Architectural Decisions**
- [x] D1–D8 all documented with rationale and constraints satisfied
- [x] Toolchain versions pinned (Vault 1.21.4, OpenTofu 1.11.0, Consul 1.22.5, Ansible-core 2.19)
- [x] Bootstrap problem explicitly resolved (D1 age key env var)
- [x] DR paths distinct and independently operable (D2/D5 + separate runbooks)

**✅ Implementation Patterns**
- [x] Naming conventions: 8 rules covering Vault paths, AppRoles, OpenTofu resources, variables, HCL files, SOPS files, Ansible variables and tasks
- [x] Structure patterns: OpenTofu module layout, Ansible layout, SOPS layout
- [x] Process patterns: AppRole onboarding checklist, state encrypt rule, break-glass token lifecycle rule

**✅ Project Structure**
- [x] Complete directory tree: 30+ named files across 4 top-level areas
- [x] Module dependency chain with targeted destroy paths documented
- [x] Provider, secrets, and module boundaries defined
- [x] FR-to-directory mapping table complete

### Architecture Readiness Assessment

**Overall Status: READY FOR IMPLEMENTATION**

**Confidence Level: High** — all 39 FRs covered, 8 explicit architectural
decisions, complete directory tree, named runbook files for all 6 operator
journeys, implementation patterns locking 10 potential agent conflict points.

**Key strengths:**
- Bootstrap problem solved explicitly (no hand-waving)
- Two DR paths architecturally distinct — not a single procedure with variants
- Isolation verification suite is a first-class MVP deliverable with a named home
- All sensitive values have explicit boundary rules (`sensitive = true`, SOPS encryption)
- Monitoring-readiness is structural — no rearchitecting when `terminus-infra-monitoring` arrives

**Deferred items (not gaps — intentional scope boundaries):**
- Automated secret rotation (Growth — manual rotation proves the discipline first)
- Consul health check integration (Growth — health endpoint ready in MVP)
- CVE automation (`terminus-infra-dependency-health` initiative)
- External PKI / Step-CA (Vision or future initiative)

### Implementation Handoff

**For implementation agents (Amelia):**
- Follow all naming patterns exactly — Vault path hierarchy and AppRole naming are invariants
- Use the AppRole onboarding checklist for every new workload — no shortcuts
- State encrypt-before-commit is mandatory — add `make encrypt-state` as first Makefile target
- Vault provider alias pattern is required for `vault-core/` init sequence — see coherence note
- Every runbook file in `docs/runbooks/` must exist before MVP is complete — not optional documentation

**First implementation step:**
```bash
export SOPS_AGE_KEY_FILE=/path/to/age-key.txt
tofu init
tofu plan
```
All implementation begins with age key export — this is the Day-0 operator briefing.
