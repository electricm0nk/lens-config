---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation-skipped, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish]
inputDocuments:
  - _bmad-output/lens-work/initiatives/terminus/infra/secrets.yaml
documentCounts:
  briefs: 0
  research: 0
  brainstorming: 0
  projectDocs: 1
workflowType: 'prd'
initiative: terminus-infra-secrets
classification:
  projectType: infrastructure_platform
  domain: operations_security
  complexity: high
  projectContext: greenfield
  criticalConstraints:
    - blast_radius_minimization
    - per_workload_secret_isolation
    - zero_trust_internal_and_external
    - openclaw_llm_compartmentalization
    - future_external_exposure_readiness
    - financial_sector_bar_pci_soc2
    - blast_and_repave_automation
    - audit_logging_and_retention
    - automated_secret_rotation
  knownDeviations:
    - control: separation_of_duties
      reason: single_operator_homelab
      compensating: all_privileged_ops_logged_policy_driven_code_reviewed_automation_no_adhoc_root_access
---

# Product Requirements Document - terminus-infra-secrets

**Author:** Todd Hintzmann
**Date:** 2026-03-23

## Executive Summary

`terminus-infra-secrets` establishes the secret management foundation for the terminus homelab ecosystem. It implements a two-layer architecture — SOPS+age for Git-native secret authoring and HashiCorp Vault KV v2 for runtime secret delivery — provisioned via OpenTofu on Proxmox VMs. The initiative targets a single operator building financial-sector-caliber security discipline from first principles, with the explicit goal of developing transferable professional skills for PCI-DSS and SOC2 regulated environments.

Every architectural decision is made consciously and documented. Automation is the discipline, not a shortcut: Day-0 bootstrap and Day-2 operations are fully scripted so the entire capability can be reproduced from git without manual intervention — equivalent to executing a DR runbook under audit conditions. Day-0 for a secrets system is architecturally distinct from all other initiatives: the bootstrap problem (secrets needed to provision the system that stores secrets) must be solved explicitly, not papered over.

The career transfer goal is concrete: upon completion, Todd can walk into a fintech design review and say "I built and operated a Vault cluster to PCI-DSS standards — automated rotation, tamper-evident audit logs, per-workload secret isolation, documented DR runbook, and a tested blast-and-repave procedure."

**Primary operator:** Todd Hintzmann (single operator — known SoD deviation, documented with compensating controls)
**Target repo:** `terminus.infra`
**Dependency position:** Foundation — all other infra and platform initiatives consume this capability before they can operate.

### What Makes This Special

This is not an installation exercise — it is a deliberate skills lab operating at enterprise standards. The homelab provides a safe environment to build and operate Vault correctly, make real architectural mistakes and recover from them, and internalize the judgment required to operate a secrets management system in a regulated financial environment.

The blast-and-repave requirement elevates every decision: if the system cannot be reproducibly rebuilt from git in an afternoon, it fails a core requirement. This forces the same reproducibility discipline that regulated environments require as formal DR evidence.

Secret compartmentalization is a first-class constraint. Per-workload identities and per-service Vault policies ensure that a compromised workload — including the OpenClaw LLM — cannot access credentials outside its own namespace. Audit logging and secret isolation are **testable controls**, not configuration states — both will have defined verification tests.

The system is designed with **monitoring-readiness as a design constraint**, not an afterthought. All Vault audit log streams must be structurally compatible with standard log aggregation and alerting pipelines (SIEM, Loki, Prometheus Alertmanager). Active monitoring and alerting is deferred to a future `terminus-infra-monitoring` initiative, but the secrets capability must expose the right signals — suspicious access patterns, failed auth attempts, secret lease anomalies — so that monitoring can be connected without rearchitecting the secrets layer. This gap is called out as an early-lifecycle priority for the terminus domain.

The system is designed for eventual external exposure without requiring security retrofits.

### Project Classification

| Field | Value |
|---|---|
| **Project Type** | Infrastructure Platform — foundation security layer |
| **Domain** | Operations / Security (financial sector bar) |
| **Complexity** | High — cross-cutting, foundational, all downstream depends on it |
| **Project Context** | Greenfield |
| **Regulatory Target** | PCI-DSS / SOC2 Type II patterns |
| **Known Deviation** | Separation of Duties (single operator; compensating: all privileged ops policy-driven, logged, code-reviewed automation only) |

### Known Gaps / Related Future Initiatives

| Gap | Scope | Priority |
|---|---|---|
| Active monitoring & alerting | `terminus-infra-monitoring` (future initiative) | High — early terminus domain lifecycle |
| External exposure hardening | Pinhole firewall patterns | Medium — design for it now, implement when needed |
| Separation of Duties | Documented deviation, single-operator compensating controls | Accepted |
| Consul role disambiguation | Service mesh vs. Vault storage backend — expected: Raft Integrated Storage + Consul for mesh only | Architecture phase |
| Dependency health & CVE automation | `terminus-infra-dependency-health` (new initiative) — OSV/NVD query against OpenTofu lock files, terminus-level patching policy document | High — needed before any component reaches first major upgrade |

---

## Success Criteria

### User Success

- OpenTofu provisions downstream initiatives by retrieving Vault secrets — zero manual credential handling
- New secret: plaintext → SOPS → git → Vault → workload readable in one documented workflow
- OpenClaw secret isolation verified via test (cannot read outside its namespace)
- Vault rebuilt from git + SOPS in one run; `tofu apply` → operational Vault, all secrets readable (blast-and-repave tested)
- Vault data layer restored from Raft snapshot with three-part acceptance test: secrets readable, AppRole auth functional, audit log resumes (snapshot restore tested)
- Todd can explain every policy boundary, auth method, and path hierarchy decision without notes

### Business Success (Skills Transfer)

- Bootstrapped Vault from scratch (Day-0 automation: single `tofu apply` → operational)
- Onboarded ≥3 distinct workload identities (OpenTofu, k3s, OpenClaw)
- Executed ≥1 planned secret rotation end-to-end
- Executed ≥1 simulated blast-and-repave (full reprovisioning from git, config layer)
- Executed ≥1 full snapshot restore (Vault data layer) with three-part acceptance test
- Can articulate: *"I built Vault to PCI-DSS patterns — AppRole identities, 90-day rotation, tamper-evident audit logs, SIEM-ready, DR tested with documented RTO"*

### Technical Success

- Vault KV v2 standalone, single Proxmox VM, provisioned via OpenTofu
- Storage backend: Integrated Storage (Raft) — snapshot-capable, no external storage dependency
- Consul role: service mesh / health checking only — **not** Vault storage backend (disambiguated in architecture phase)
- SOPS+age authoring pipeline; age key custody documented
- Per-workload AppRole identities; zero shared credentials
- Vault audit log → structured file (SIEM/Loki-compatible)
- Secret TTL ≤ 90 days; automated rotation tested
- OpenClaw isolated namespace with deny-all-other-paths policy — tested
- Day-0 bootstrap: single `tofu apply` → operational Vault
- Break-glass procedure documented, access-logged, tested ≥1 time
- `vault operator raft snapshot save` procedure documented, tested, restore verified
- RTO ≤ 4 hours (snapshot restore + OpenTofu reprovision, single operator, documented and tested)
- SoD deviation formally documented with compensating controls

### Measurable Outcomes

| Outcome | Target |
|---|---|
| Bootstrap (tofu apply → Vault live) | ≤ 60 min |
| Secret delivery latency | < 2s |
| Audit coverage | 100% reads in log within 30s |
| Secret TTL | All ≤ 90 days, automated rotation tested |
| OpenClaw isolation | 0 successful reads outside its namespace |
| Blast-and-repave RTO | Documented target; full reprovisioning from git tested |
| Snapshot restore RTO | ≤ 4 hours; three-part acceptance test passed |

---

## Functional Requirements

### Secret Storage & Delivery

- **FR1:** Operator can store an encrypted secret in git using SOPS+age without plaintext ever touching the repository
- **FR2:** Operator can sync a SOPS-encrypted secret from git into Vault KV v2 at a defined path hierarchy
- **FR3:** Authorized workload can retrieve a secret from Vault KV v2 using its AppRole identity
- **FR4:** Operator can retrieve a specific version of a secret from Vault KV v2 (version history)
- **FR5:** Operator can soft-delete a secret from Vault KV v2 without destroying version history

### Identity & Access Control

- **FR6:** Operator can provision a new AppRole identity for a workload via OpenTofu (policy-as-code)
- **FR7:** Operator can scope an AppRole identity to a minimum set of Vault paths via a HCL policy committed to git
- **FR8:** Operator can revoke an AppRole secret-id without affecting other workload identities
- **FR9:** Operator can deliver an AppRole secret-id to a workload via wrapped token (single-use delivery)
- **FR10:** System enforces default-deny — any path not explicitly granted in policy returns 403
- **FR11:** Operator can verify that a workload identity cannot read outside its scoped namespace (isolation test)

### Provisioning & Automation

- **FR12:** Operator can provision a complete, operational Vault instance from zero via a single `tofu apply`
- **FR13:** Operator can reprovision Vault entirely from git after full VM destruction (blast-and-repave) with no manual steps
- **FR14:** Operator can apply Vault configuration changes (policy updates, new mounts, new AppRoles) via `tofu apply` without manual Vault CLI interaction
- **FR15:** Operator can run Ansible playbooks for Day-2 operations (secret rotation, cert renewal) from git
- **FR16:** OpenTofu can retrieve secrets from Vault during infrastructure provisioning runs using its AppRole identity

### Audit & Compliance

- **FR17:** System records every secret read, write, auth attempt, and denial to a structured audit log
- **FR18:** Operator can verify audit log contains an entry for a specific operation within 30 seconds of the operation
- **FR19:** Operator can confirm audit log resumes after a Vault restart or restore (no silent gap)
- **FR20:** Operator can produce a record of all privileged operations (break-glass use, root token activity, policy changes) from the audit log
- **FR21:** Operator can document a known compliance deviation with compensating controls in a formal deviation record

### Secret Lifecycle Management

- **FR22:** Operator can define a TTL for a secret or AppRole credential (maximum 90 days)
- **FR23:** Operator can execute a manual secret rotation end-to-end (old credential revoked, new credential issued, workload verified)
- **FR24:** System can automatically rotate a secret before its TTL expires (automated rotation — Growth)
- **FR25:** Operator can document and execute an age key rotation, re-encrypting all SOPS-managed files
- **FR26:** Operator can permanently destroy a secret and its SOPS source file per a documented destruction procedure

### Disaster Recovery & Resilience

- **FR27:** Operator can take a Vault Raft snapshot manually on demand
- **FR28:** System takes Vault Raft snapshots automatically on a defined schedule; snapshots stored off-VM
- **FR29:** Operator can restore Vault from a Raft snapshot and verify recovery via three-part acceptance test (secrets readable, AppRole auth functional, audit log active)
- **FR30:** Operator can execute the full blast-and-repave procedure and verify via three-part acceptance test
- **FR31:** Operator can access Vault using the break-glass procedure when normal authentication paths are unavailable
- **FR32:** Operator can rotate and revoke the root token after break-glass use, leaving no persistent root credential

### Observability & Health

- **FR33:** External tools can query Vault health status via the `/v1/sys/health` endpoint
- **FR34:** Operator can verify Vault seal status at any time via CLI
- **FR35:** Audit log format is structured JSON, compatible with SIEM and Loki log aggregation pipelines without transformation
- **FR36:** Consul can perform health checks against the Vault service (mesh integration — Growth)

### Dependency & Patch Management

- **FR37:** Operator can determine the currently pinned version of each security-critical component (Vault, SOPS, age) from OpenTofu lock files without consulting a separate manifest
- **FR38:** Operator can document a version exception (pinned below latest, reason stated) as a comment in the OpenTofu module and in a running exceptions log
- **FR39:** Operator can query CVE databases (OSV/NVD) for known vulnerabilities against pinned component versions (automated — `terminus-infra-dependency-health` initiative)

---

## Non-Functional Requirements

### Security

- **NFR1:** All secrets at rest encrypted — Vault Raft storage uses AES-256-GCM; SOPS plaintext never persists on disk
- **NFR2:** All secrets in transit encrypted — Vault listener requires TLS; no plaintext HTTP accepted
- **NFR3:** All Vault API authentication uses short-lived tokens derived from AppRole; no long-lived static tokens in workload configuration
- **NFR4:** Vault root token must not persist after bootstrap; any root token use must be followed by rotation/revocation within the same session
- **NFR5:** Every AppRole policy must be reviewable as a committed HCL file in git — no policy exists only in Vault's state
- **NFR6:** age private key must never be stored on a network-accessible system unencrypted; custody location documented

### Reliability

- **NFR7:** Vault availability target: best-effort (homelab, single node, planned maintenance acceptable); no SLA required
- **NFR8:** Vault unavailability must not cause data loss for secrets already stored — Raft storage is durable; loss only possible if VM is destroyed without a snapshot
- **NFR9:** All downstream workloads must degrade gracefully when Vault is unreachable (cached credentials where possible, documented fallback behavior)
- **NFR10:** Vault Raft snapshots must be stored on a different physical host than the Vault VM — snapshot loss and VM loss must not be correlated failures

### Performance

- **NFR11:** Vault KV v2 secret read latency: < 2 seconds under normal operating conditions
- **NFR12:** Day-0 bootstrap time (`tofu apply` from zero to operational Vault): ≤ 60 minutes
- **NFR13:** Snapshot restore to operational Vault (RTO): ≤ 4 hours including verification

### Operability

- **NFR14:** All operational procedures (break-glass, blast-and-repave, key rotation, secret authoring, workload onboarding) must be documented as step-by-step runbooks executable without prior context — a returning operator after 6 months must be able to operate the system using documentation alone
- **NFR15:** No manual Vault configuration permitted outside of documented break-glass scenarios — all changes via OpenTofu or documented Ansible playbooks
- **NFR16:** `tofu plan` outputting no changes is the correct steady state — any drift must be detectable and correctable via plan/apply
- **NFR17:** All operational runbooks stored in git alongside the code they operate

### Recoverability

- **NFR18:** Blast-and-repave (full reprovisioning from git) must be executable as a single `tofu apply` with no manual steps beyond providing the age key
- **NFR19:** Snapshot restore procedure must be documented, tested at least once, and produce a verified RTO measurement
- **NFR20:** Recovery procedures must be tested on a defined schedule — not assumed to work without verification

### Maintainability

- **NFR21:** Vault and all security-critical components (SOPS, age) must have pinned versions in OpenTofu lock files — no floating version constraints
- **NFR22:** Component upgrades must be applied via `tofu plan` → review → `tofu apply` — no in-place package upgrades on running VMs for security components
- **NFR23:** All version exceptions (pinned below latest) must be documented with reason and target resolution version

---

## User Journeys

### Journey 1: Day-0 Bootstrap — "The Empty Lab"

**Persona:** Todd, Platform Builder. Fresh Proxmox VM, no Vault, no secrets. Every downstream initiative is blocked until this works.

**Opening Scene:** Todd sits down with a new `terminus-infra` branch checkout. No Vault process is running anywhere. OpenTofu knows nothing about secrets yet. The k3s cluster can't start without database credentials it doesn't have. This is ground zero.

**Rising Action:** Todd runs `tofu init && tofu plan` against the Vault module. OpenTofu needs the one secret it can't store in Vault — the unseal key / root token bootstrap credential. This is the bootstrap problem: the chicken-and-egg moment every secrets operator must solve consciously. The age private key lives outside Vault, under Todd's physical control. SOPS decrypts the bootstrap material; OpenTofu provisions the Vault VM, configures auth methods, mounts KV v2, writes initial policies — all from code, no manual steps.

**Climax:** `tofu apply` completes. Vault is unsealed, initialized, configured. OpenTofu immediately verifies its own access — it reads a test credential using its AppRole identity. The read succeeds. The system exists and trusts itself.

**Resolution:** Todd pushes a small config change downstream: a k3s module now references Vault instead of a hardcoded credential. The next `tofu apply` on that module returns a live secret from Vault. The foundation is load-bearing. Elapsed time: under 60 minutes from zero to operational.

**Reveals requirements for:** VM provisioning, Vault init/unseal automation, SOPS bootstrap credential decryption, AppRole self-verification, Day-0 idempotency.

---

### Journey 2: Secret Authoring — "Committing a Secret Without Fear"

**Persona:** Todd, Operator. A new database password needs to go into the ecosystem. The wrong path: paste it into a config file and commit. The right path: this journey.

**Opening Scene:** Todd generates a strong credential for the Postgres `terminus_app` role. It exists only in his terminal buffer. He needs it in Vault KV v2 at `secret/terminus/infra/postgres/app-role`, accessible to OpenTofu, never in plain text in git.

**Rising Action:** Todd opens the SOPS-encrypted secrets file for the Postgres subsystem. He adds the new credential, saves, and SOPS re-encrypts via age key automatically. The diff shows only ciphertext changing — the plaintext never touches disk unencrypted outside the editor session. Todd commits the encrypted file to git. He then runs the Vault sync playbook: SOPS decrypts locally, `vault kv put` writes to the correct path with the correct policy boundary.

**Climax:** Todd verifies from the OpenTofu AppRole: `vault kv get secret/terminus/infra/postgres/app-role` returns the credential. He verifies from the OpenClaw AppRole that the same path is **not** readable — access denied, as expected. The audit log shows both reads: one permitted, one denied.

**Resolution:** The credential is in Vault, in git (encrypted), and audited. Todd closes his terminal. The plaintext never persisted.

**Reveals requirements for:** SOPS+age authoring workflow documentation, KV v2 path hierarchy, per-policy read verification, audit log entry verification.

---

### Journey 3: Workload Identity Onboarding — "Giving k3s a Voice"

**Persona:** Todd, Platform Integrator. k3s is running but has no Vault identity. Time to give it one.

**Opening Scene:** A k3s workload needs to pull a TLS certificate and a database DSN from Vault. Right now it has no Vault identity. Todd needs to create an AppRole, scope a policy, and deliver credentials to k3s without ever hardcoding a token.

**Rising Action:** Todd writes a Vault policy file: `k3s-workload.hcl` — read-only on `secret/terminus/infra/k3s/*`, deny everything else. Applied via OpenTofu (policy-as-code, reviewable, in git). He creates an AppRole with a wrapped secret-id delivery. The Vault Agent sidecar in k3s receives the role-id via ConfigMap and the secret-id via the wrapped delivery mechanism — consumed once, never stored in plaintext.

**Climax:** The k3s workload starts, Vault Agent authenticates, injects secrets into the pod environment. Todd verifies: the OpenClaw AppRole cannot read `secret/terminus/infra/k3s/*`. The boundary holds.

**Resolution:** Workload identity #2 onboarded. The pattern is repeatable — next workload is a copy-modify of the same policy template. Zero credential exchanges that weren't logged.

**Reveals requirements for:** AppRole lifecycle (create, rotate, revoke), policy templating, Vault Agent sidecar pattern, wrapped secret-id delivery, per-namespace isolation enforcement.

---

### Journey 4: Break-Glass — "3am, Vault Is Locked"

**Persona:** Todd, Incident Responder. Vault sealed after VM reboot. Everything downstream is degraded.

**Opening Scene:** Vault starts sealed after a Proxmox host reboot. Auto-unseal has failed. k3s workloads are returning 503s. OpenTofu can't run. Todd is staring at a sealed Vault.

**Rising Action:** Todd opens the break-glass runbook. Retrieving the runbook itself is logged — the act of access triggers an audit trail. He retrieves the unseal keys from their documented offline custody location. Runs `vault operator unseal` with the required key shares. Each unseal attempt is logged. Vault unseals. Todd verifies health and confirms one downstream secret is readable.

**Climax:** Todd rotates the root token immediately per runbook — the break-glass token is single-use. He commits a note to the ops log: timestamp, what happened, what was done, root token rotation confirmed. The audit log shows the full sequence.

**Resolution:** Vault is operational. Downstream workloads recover as Vault Agents re-authenticate. Total time from alert to resolution: under 20 minutes. Follow-up ticket: configure auto-unseal.

**Reveals requirements for:** Break-glass runbook (documented step-by-step), root token rotation procedure, unseal key custody documentation, audit log continuity through unsealing, auto-unseal consideration.

---

### Journey 5a: Blast-and-Repave — "Starting From Zero on Purpose"

**Persona:** Todd, DR Executor. Scheduled destruction of the Vault VM. Tests that the config layer is fully reproducible from git — no snapshot involved.

**Opening Scene:** Todd destroys the Vault VM entirely: `tofu destroy -target vault_vm`. Vault is gone. This tests that git + SOPS is the complete source of truth for config and secrets.

**Rising Action:** `tofu apply`. OpenTofu provisions a new VM, installs Vault, runs init, configures auth methods, mounts KV v2, writes all policies, registers all AppRoles. SOPS-encrypted secrets are re-injected from git. No manual steps.

**Climax:** Three-part verification: OpenTofu AppRole reads its secrets ✅. k3s AppRole reads its secrets ✅. OpenClaw AppRole denied outside its namespace ✅. Audit log active ✅.

**Resolution:** Elapsed time noted against the ≤ 60 min bootstrap target. Gap surfaced: any secret written directly to Vault without a SOPS-encrypted git source is unrecoverable. Discipline enforced: every secret in Vault must have a SOPS source in git.

**Reveals requirements for:** Full reprovisioning idempotency, SOPS-as-source-of-truth discipline, bootstrap RTO measurement, three-part verification suite.

---

### Journey 5b: Snapshot Restore — "Getting the Data Back"

**Persona:** Todd, DR Executor. Vault storage backend corrupted; VM and config layer intact. Tests the Raft snapshot recovery path.

**Opening Scene:** Todd simulates data corruption. He runs `vault operator raft snapshot restore vault-backup-$(date +%Y%m%d).snap`. Vault restores from snapshot and unseals.

**Climax:** Three-part acceptance test:
1. `vault kv get secret/terminus/infra/postgres/app-role` — returns credential ✅
2. OpenTofu AppRole issues a token successfully ✅
3. Audit log shows an active entry within 30 seconds ✅

**Resolution:** RTO logged against the ≤ 4 hour target. Blast-and-repave and snapshot restore are confirmed as two distinct drills with two distinct purposes — both documented separately in the runbook.

**Reveals requirements for:** Raft snapshot automation (scheduled), off-VM snapshot storage, restore procedure, RTO measurement, backup retention policy.

---

### Journey 6: OpenClaw Isolation Test — "The Machine That Must Not Know"

**Persona:** OpenClaw LLM service, automated consumer. The adversarial consumer journey — testing the boundary, not the happy path.

**Opening Scene:** OpenClaw is configured with its AppRole. Policy grants read access to `secret/terminus/agent/openclaw/*` only. The test verifies a compromised or misconfigured OpenClaw cannot pivot to other namespaces.

**Rising Action:** Automated verification suite runs using OpenClaw's token: attempts to read `secret/terminus/infra/postgres/app-role`, `secret/terminus/infra/k3s/*`, and `sys/policy`. Each attempt returns `403 permission denied`.

**Climax:** Zero successful reads outside the OpenClaw namespace. Every attempt is in the audit log. This pattern becomes the "normal" baseline — monitoring can alert on any deviation.

**Resolution:** OpenClaw's blast radius is confirmed bounded. Even if the LLM inference process is exploited, credential exposure is limited to its own namespace. This test runs as part of every Vault reprovisioning verification.

**Reveals requirements for:** OpenClaw deny-all policy (outside namespace), automated isolation verification suite, denied-access audit log entries as expected signal.

---

### Journey Requirements Summary

| Journey | Key Capabilities Revealed |
|---|---|
| Day-0 Bootstrap | VM provisioning, Vault init/unseal automation, SOPS bootstrap, AppRole self-verify, idempotency |
| Secret Authoring | SOPS authoring workflow, KV v2 path hierarchy, cross-policy read verification, audit entry check |
| Workload Identity Onboarding | AppRole lifecycle, policy templating, Vault Agent sidecar, wrapped secret-id, namespace isolation |
| Break-Glass | Break-glass runbook, root token rotation, unseal key custody, audit continuity |
| Blast-and-Repave | Full reprovisioning from git, SOPS-as-source-of-truth discipline, RTO measurement |
| Snapshot Restore | Raft snapshot automation, off-VM storage, three-part acceptance test, retention policy |
| OpenClaw Isolation | Deny-all policy outside namespace, automated isolation verification, denied-access as audit baseline |

---

## Domain-Specific Requirements

### Compliance & Regulatory

| Standard | Applicability | Posture |
|---|---|---|
| PCI-DSS v4.0 (Req 3, 8, 10) | Cryptographic key management, identity/access, audit logging | Training target — build to pattern, document deviations |
| SOC 2 Type II (CC6, CC7) | Logical access, system operations, change management | Training target — evidence artifacts produced, not formally audited |
| NIST SP 800-57 | Cryptographic key lifecycle (generation, storage, rotation, destruction) | Applied — age key custody documented, rotation procedure defined |
| CIS Vault Benchmark | Vault hardening baseline | Applied where compatible with standalone/homelab constraints |

**Regulatory posture:** This is a homelab skills lab, not a production system under audit. The target is to build to PCI-DSS / SOC2 *patterns* such that the skills, artifacts, and judgment produced are directly transferable to a regulated environment. No formal audit is in scope.

---

### Technical Constraints

**Encryption & Key Management**
- All secrets at rest: Vault storage encrypted (Raft with AES-256-GCM)
- All secrets in git: SOPS+age encrypted; plaintext never committed
- age private key: single-custodian (Todd), custody documented (location, rotation procedure, recovery scenario)
- age key rotation: defined procedure; rotation triggers re-encryption of all SOPS-managed files

**Access Control**
- Zero shared credentials — all workload authentication via per-identity AppRole
- Default deny: all policies start from `deny` baseline; capabilities granted explicitly
- Least-privilege: each AppRole scoped to minimum required paths
- No persistent root tokens: root token rotated/revoked after bootstrap; break-glass token single-use

**Audit & Observability**
- Vault audit log: structured JSON file, every read/write/auth/denial captured
- Audit log location: local file on Vault VM (SIEM/Loki-compatible format)
- Audit log integrity: file on same VM — tamper-evidence is a **known accepted risk** in homelab context; mitigated by single-operator scope and future log-shipping to `terminus-infra-monitoring`
- Audit log retention: minimum 90 days local; archival policy TBD (architecture phase)
- All privileged operations (break-glass, root token use, policy changes): logged, documented in ops log

**Secret Lifecycle**
- Secret TTL: ≤ 90 days (PCI-DSS Req 8.3 pattern)
- Rotation: automated where possible; manual rotation procedure documented for all secrets
- Revocation: AppRole secret-id revocation procedure documented and tested
- Destruction: SOPS file deletion procedure included in secret lifecycle documentation

---

### Known Deviations (Domain Layer)

| Control | Standard | Reason | Compensating Control |
|---|---|---|---|
| Separation of Duties | PCI-DSS Req 8 / SOC2 CC6 | Single operator | All privileged ops logged; policy-driven automation; no ad-hoc root access |
| Dual-control for key ceremonies | NIST SP 800-57 | Single custodian | age key custody fully documented; key material stored in physically separate offline location; key rotation procedure defined and tested |
| Tamper-evident remote audit log | PCI-DSS Req 10.5 | Local file only (homelab) | Accepted risk; structured format ensures future log-ship to `terminus-infra-monitoring` requires no rearchitecting |

---

### Risk Register

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| age private key loss | Critical — all SOPS material unrecoverable | Low | Documented custody; physically separate backup; rotation procedure tested |
| Vault storage corruption without snapshot | High — data layer unrecoverable | Low | Raft snapshot automated; off-VM storage; restore tested |
| Overly broad AppRole policy | High — lateral movement possible | Medium | Policy review as part of every onboarding; isolation tests automated |
| OpenClaw LLM compromise | High — blast radius if policy misconfigured | Medium | Namespace isolation; automated isolation verification runs on every reprovision |
| Audit log gap during unsealing | Medium — compliance blind spot | Low | Break-glass procedure includes audit log verification step post-unseal |

---

## Infrastructure Platform Requirements

### Platform Overview

`terminus-infra-secrets` is an operator-facing infrastructure service: provisioned and managed via IaC, consumed programmatically by downstream workloads via the Vault API. There is no end-user interface. All interaction paths are either operator → Vault (via OpenTofu, Vault CLI, Ansible) or workload → Vault (via Vault API with AppRole auth).

### Consumer Integration Interface

| Consumer | Integration Method | Auth |
|---|---|---|
| OpenTofu (infra provisioning) | Vault Terraform provider (`hashicorp/vault`) | AppRole |
| k3s workloads | Vault Agent sidecar — injects secrets to pod filesystem | AppRole |
| OpenClaw LLM | Vault Agent or direct API read | AppRole (isolated namespace) |
| Future consumers | Same AppRole pattern — one identity per workload | AppRole |

**Interface contract:** Consumers reference secrets by Vault path (`secret/terminus/{domain}/{service}/{key}`). Path hierarchy is stable after initial provisioning — path renames are a breaking change requiring coordinated migration.

### Operational Management Interface

| Operation | Interface | Notes |
|---|---|---|
| Provisioning | OpenTofu (`tofu apply`) | All config-as-code |
| Policy authoring | HCL files in git → `tofu apply` | No manual `vault policy write` |
| Secret authoring | SOPS-encrypted files → Vault sync playbook | No plaintext in git |
| Vault CLI | Break-glass + verification only | Not primary management interface |
| Vault UI | **Out of scope** (MVP) | Increases attack surface; not needed |
| Ansible | Day-2 operations (rotation, cert renewal) | Playbooks in git |

### Configuration Management

- All Vault configuration declarative in OpenTofu modules — no configuration drift permitted
- Policy files (`.hcl`) committed to git and applied via OpenTofu — never applied manually
- Vault version pinned in OpenTofu module; upgrades via module version bump → `tofu plan` → review → `tofu apply`
- `tofu plan` with no changes is the correct steady state

### Health & Readiness Signals

| Signal | Method | Consumer |
|---|---|---|
| Vault health | `GET /v1/sys/health` | Consul health check, future monitoring |
| Vault seal status | `vault status` | Break-glass runbook, blast-and-repave verification |
| Audit log active | Log file last-modified timestamp | Future monitoring alerting |
| AppRole auth reachable | Token issue test in verification suite | Post-reprovision verification |

### Upgrade & Patching

- **Vault, SOPS, age** (security-critical): 30-day cool-down after minor release before applying, unless patching a known CVE
- **OpenTofu, Ansible, Consul** (application components): 60-day cool-down after minor release
- **Critical CVE**: apply as soon as a patched version is available and tested — no cool-down
- **Exception mechanism**: one-line comment in the OpenTofu module pinning the version + entry in a running exceptions log (e.g., `# pinned at 1.21.4 pending CVE-2026-XXXX fix in 1.21.5`)
- **Version source of truth**: OpenTofu lock files and `required_providers` blocks — no separate version manifest; versions are read from existing IaC files
- **CVE query automation**: deferred to `terminus-infra-dependency-health` initiative — will query OSV/NVD against pinned versions from OpenTofu lock files
- Immutable patching preferred: repave VM rather than patch-in-place where practical
- No zero-downtime requirement (homelab, single operator); planned maintenance window acceptable

### Implementation Constraints

- Vault KV v2 (not v1) — soft-delete, versioning, metadata support
- AppRole auth method only in MVP — no userpass, LDAP, or GitHub auth
- No Vault namespaces (Enterprise feature) — path hierarchy enforces isolation in OSS
- `VAULT_ADDR` and `VAULT_TOKEN` never in workload environment variables — Vault Agent injection only
- TLS on Vault listener required (self-signed CA acceptable for homelab; documented)

### Dependency & Patch Management Policy

| Component | Tier | Cool-down | CVE Response |
|---|---|---|---|
| Vault | Security-critical | 30 days | Immediate |
| SOPS | Security-critical | 30 days | Immediate |
| age | Security-critical | 30 days | Immediate |
| OpenTofu | Application | 60 days | Immediate |
| Consul | Application | 60 days | Immediate |
| Ansible-core | Application | 60 days | Immediate |

**Policy reference:** Full terminus-level patching policy governed by `terminus-infra-dependency-health` initiative (future). This initiative operates under the policy defined above until that initiative delivers the authoritative governance document.

---

## Product Scope

### MVP Strategy & Philosophy

**MVP Approach:** Platform MVP — the minimum that makes the foundation load-bearing. No downstream initiative can operate without authenticated secret delivery. MVP is complete when OpenTofu can provision a downstream resource by retrieving a secret from Vault with zero manual credential handling.

**MVP is NOT:** a demo, a prototype, or a partial implementation. It is a production-pattern Vault instance that can be destroyed and rebuilt from git in an afternoon.

**Scope principle:** If it can be done manually and documented without compromising the security posture, it can be deferred to Growth. If its absence creates a security gap or makes the system unreproducible, it is MVP.

---

### Phase 1: MVP

**Exit criterion:** `tofu apply` → Vault operational → OpenTofu reads a live secret → blast-and-repave tested → snapshot restore tested → break-glass documented.

| Capability | Justification |
|---|---|
| Vault KV v2 standalone (Raft, single VM) | Foundation — nothing works without this |
| SOPS+age authoring pipeline | Git-safe secret authoring — required before any secret touches git |
| AppRole for OpenTofu | First consumer; proves the integration pattern |
| Vault policies: default-deny + OpenTofu read scope | Security posture — no policy = no isolation |
| TLS on Vault listener (self-signed CA) | Required — plaintext Vault listener not acceptable even in homelab |
| Vault audit log → structured JSON file | PCI-DSS pattern; required before any secret is written |
| Day-0 bootstrap automation (`tofu apply` → operational) | Core requirement — manual bootstrap is not reproducible |
| Break-glass procedure (documented + tested ≥1 time) | Required before any real secrets are stored |
| Blast-and-repave test (config layer) | Required — validates git as source of truth |
| Snapshot restore test (data layer, three-part acceptance) | Required — validates data recovery path |
| age key custody documentation | Required — key loss = unrecoverable SOPS material |
| SoD + dual-control deviations formally documented | Required — compensating controls must exist before operation |

---

### Phase 2: Growth

**Entry criterion:** MVP exit criterion passed. Secrets foundation is operational and DR-tested.

**Sequencing:** k3s AppRole and OpenClaw AppRole are independent and can proceed in parallel. Consul mesh integration is additive and does not block either. Automated rotation can begin after at least one manual rotation has been executed and documented.

| Capability | Depends On | Notes |
|---|---|---|
| AppRole for k3s workloads | MVP | Second consumer; validates pattern at scale |
| AppRole for OpenClaw + isolation policy | MVP | High-priority — LLM threat vector; isolation test automated |
| OpenClaw isolation verification suite | OpenClaw AppRole | Automated; runs on every reprovision |
| Vault Agent sidecar pattern (k3s) | k3s AppRole | Eliminates env-var credential injection in pods |
| Consul integration (service mesh / health checking) | MVP | Consul role disambiguated — mesh only, not Vault storage |
| Automated secret rotation (all components) | Manual rotation executed ≥1 time | Automation after discipline is proven manually |
| `terminus-infra-dependency-health` initiative | MVP complete | CVE query against OpenTofu lock files; patching policy governance |

---

### Phase 3: Vision

**Entry criterion:** Growth capabilities operational. Vault has been through ≥1 rotation cycle and ≥1 unplanned recovery scenario.

| Capability | Depends On | Notes |
|---|---|---|
| `terminus-infra-monitoring` connected (Vault → Loki → Alertmanager) | terminus-infra-monitoring initiative | Secrets layer is monitoring-ready by design; no rearchitecting needed |
| External exposure hardening (mTLS, pinhole firewall) | Monitoring connected | Don't expose externally until monitoring is watching |
| Formal secret lifecycle runbook as audit artifact | All Growth capabilities | PCI-DSS / SOC2 evidence artifact; requires full lifecycle to be documented |

---

### Risk-Based Scoping

| Risk | Mitigation | Scope Impact |
|---|---|---|
| Day-0 bootstrap complexity underestimated | Bootstrap problem scoped explicitly in MVP; Winston flags in architecture phase | None — already in MVP |
| OpenClaw isolation misconfiguration | Isolation test suite required before Growth exit | Growth exit criterion |
| age key loss before custody documented | Custody documentation is MVP — no real secrets until this exists | MVP blocker |
| Vault version upgrade breaks AppRole consumers | 30-day cool-down + tested upgrade path via `tofu plan` | Growth operational concern |
| Scope creep from Vault Enterprise features | Explicitly out of scope — OSS only; Vault namespaces deferred | Governance |

---

### Out of Scope (All Phases)

- Vault Enterprise features (namespaces, HSM auto-unseal, DR replication)
- Vault UI (increases attack surface; CLI + API only)
- Multi-operator / LDAP / SSO authentication
- Formal PCI-DSS or SOC2 audit engagement
- External PKI / Vault PKI secrets engine (future initiative if needed)
