---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish, step-12-complete]
inputDocuments:
  - docs/terminus/infra/postgres/businessplan-questions.md
workflowType: prd
initiative: terminus-infra-postgres
phase: businessplan
mode: batch
date: "2026-03-24"
author: CrisWeber
---

# Product Requirements Document — terminus-infra-postgres

**Author:** CrisWeber
**Date:** 2026-03-24
**Initiative:** terminus-infra-postgres
**Track:** feature
**Phase:** BusinessPlan
**Target Repo:** terminus.infra

---

## Executive Summary

The `terminus-infra-postgres` initiative establishes a highly available PostgreSQL cluster on dedicated Proxmox VMs as a shared infrastructure capability for the Terminus homelab. It is a foundational greenfield build — nothing in the platform layer can be deployed until a reliable, secure, and recoverable database substrate exists.

The core insight driving this feature: PostgreSQL is not just a database install — it is a long-lived operational product with its own availability contract, credential lifecycle, backup discipline, security posture, and blast-and-repave story. Every decision made here either enables or constrains everything that runs on top of it.

This feature is deliberately designed as a skills-transfer platform: the patterns, automation, and operational discipline applied here are directly transferable to production fintech environments. Security comes first. All changes go through git. No credentials touch plaintext storage.

---

## Problem Statement

### The Real Problem

Terminus needs a robust, repeatable, and operationally sound database substrate before k3s, Temporal, or any other platform service can be deployed safely. Without it:

- k3s cannot be provisioned (postgres is a required dependency)
- Temporal has no persistence layer
- There is no safe place to host application databases as new services are developed
- No skills-transfer substrate exists for practicing production-grade database operations

Ad hoc database installations are not acceptable — they are hard to reprovision, have no credential discipline, no backup story, and no HA.

### Why Now

k3s is blocked on postgres. Everything after k3s in the `dependency_order` — Crossplane, Temporal, platform features — is also blocked. Postgres must be delivered before any of those initiatives can reach implementation.

### Why This Approach

- **Dedicated VMs on Proxmox** — not in k3s (which doesn't exist yet) and not co-located with other services. Isolation and independent lifecycle.
- **HA cluster from day 0** — single-node postgres is a false economy. Downtime during development causes cascading failures. HA is simpler to design in than retrofit.
- **OpenTofu as provisioning authority** — aligns with infra architecture principles. Blast-and-repave must work from git alone.
- **Vault-managed credentials** — consistent with the secrets foundation already built. No plaintext passwords anywhere.
- **Security-first posture** — homelab fintech context; patterns applied here must be production-transferable.

---

## Success Criteria

### Operator Success

1. Operator can access the PostgreSQL management UI and verify cluster status
2. Operator can create a new database with a dedicated service role via automation; credentials are automatically stored in Vault KV v2
3. Operator can stand up a test database, insert test data, and tear it down — fully automated
4. Operator can perform a manual backup on demand and verify it completes successfully
5. All provisioning, database creation, backup, and restore operations are visible in PostgreSQL audit logs

### Operational Health (6-Month View)

- Daily automated backups are running and verified
- Cluster HA is verified and documented — a primary node failure triggers failover without operator intervention
- Any version upgrade or configuration change is delivered via the standard PR workflow with no manual VM changes

> **Note:** Monitoring and alerting are intentionally not a criterion of this feature. This is a greenfield ecosystem; observability will be delivered by a dedicated future monitoring initiative scoped after the critical substrate (postgres, k3s) is operational. The postgres cluster must be designed to *accept* monitoring integration (structured JSON audit logs — NFR12; metrics endpoint compatible with Prometheus — NFR noted as growth phase) but does not own delivery of the monitoring solution. For MVP, FR20 audit log verification — confirming that structured audit logs are active and producing data — is the sufficient observability signal.

### Definition of Done

The feature is `done` when all five Operator Success criteria above are demonstrably satisfied by automated acceptance tests, and the blast-and-repave procedure has been executed end-to-end at least once.

### Failure Modes to Avoid

- Any credential committed to git in plaintext
- No documented backup strategy or untested restore procedure
- No runbook for blast-and-repave
- Manual VM changes outside of documented procedures
- A single VM loss that causes unrecoverable data loss
- Cluster sizing that starves other Proxmox services

---

## Operator Journeys

### Journey 1 — Day-0: Initial Cluster Provisioning

**Persona:** Todd (sole operator), standing up a fresh Terminus homelab environment.

Todd has Proxmox running and the secrets infrastructure bootstrapped (Vault operational, age key in custody). He runs `tofu apply` from the root of `terminus.infra`. OpenTofu provisions two (or more) Proxmox VMs, bootstraps the PostgreSQL HA cluster via Ansible, configures TLS, creates the superadmin role, initializes the replication topology, and stores the admin credentials in Vault. When `tofu apply` finishes, Todd opens the management UI and sees the cluster — primary and replica — both green. He has never touched a VM directly.

*Success moment:* `SELECT pg_is_in_recovery();` returns `f` on primary, `t` on replica. Cluster is operational.

---

### Journey 2 — Day-1: Onboarding a New Database (e.g., for Temporal)

**Persona:** Todd, provisioning a postgres database for a new platform service.

Todd adds a new database definition to the OpenTofu config (or runs an Ansible playbook — exact mechanism TBD in techplan). Automation creates the database, creates a service role scoped to that database only, generates credentials, and stores them in Vault at `secret/terminus/{env}/postgres/{dbname}/...`. Todd verifies the credentials land in Vault and that the service role cannot access any other database.

*Success moment:* Service can authenticate to its dedicated database; cannot read or write to any other.

*Note:* The exact tooling for database/role provisioning (OpenTofu resource vs. Ansible playbook vs. documented script) is a techplan decision (see Open Questions).

---

### Journey 3 — Day-2: Routine Operations

**Persona:** Todd, maintaining the cluster over time.

Primary routine tasks are:
- **Patch / version upgrades** — PR with version pin bump → plan → apply
- **Backup verification** — periodic manual restore test to confirm backup integrity
- **Credential rotation** — documented runbook, Vault-assisted
- **Schema migrations** — owned by consuming service (exact tooling TBD, possibly Liquibase)

*Success moment:* Every operational action has a runbook. Nothing requires remembering undocumented steps.

---

### Journey 4 — Break-Glass / Disaster Recovery

**Persona:** Todd, responding to a complete VM loss.

A Proxmox VM hosting a cluster node is lost. Todd:
1. Verifies cluster HA has already promoted the replica (automated failover)
2. If both nodes are lost, pulls backups from the off-VM backup host
3. Runs `tofu apply` to reprovision the missing VM(s)
4. Restores from backup using the documented restore runbook
5. Verifies cluster integrity with the three-part acceptance test (cluster healthy, application data readable, audit log active)

*RTO target: ≤ 4 hours from incident to verified operational cluster.*

*Success moment:* Three-part acceptance test passes. Logs show uninterrupted audit trail.

---

## Functional Requirements

### Provisioning

**FR1:** Operator can provision a complete, operational HA PostgreSQL cluster from zero via a single `tofu apply` with no manual VM steps.

**FR2:** Operator can reprovision the cluster entirely from git after full VM destruction (blast-and-repave) with no manual steps beyond providing the age key.

**FR3:** Operator can provision a new PostgreSQL database with a dedicated service role via automation; the process requires no direct VM access.

**FR4:** Newly provisioned database credentials are automatically stored in Vault KV v2 at `secret/terminus/{env}/postgres/{dbname}/...` with the following minimum field schema: `username`, `password`, `host`, `port`, `db_name`, `ssl_mode`. All minimum fields must always be populated by provisioning automation; this is the consumer credential contract.

**FR5:** Operator can retrieve database credentials from Vault KV v2 using the established secrets infrastructure.

### Access & Authentication

**FR6:** Each database has a dedicated service role scoped to that database only; no application workload uses a superuser account.

**FR7:** Operator can access a PostgreSQL management UI for monitoring, verification, and authorized ad-hoc queries.

**FR8:** PostgreSQL accepts TLS connections only; plaintext connections are rejected at the listener.

**FR9:** Operator can revoke a service role credential and issue a new one without affecting other database identities.

### High Availability

**FR10:** PostgreSQL cluster provides high availability such that a single node failure causes no data loss and no service interruption requiring operator intervention.

**FR11:** Operator can verify cluster HA status (primary, replica health, replication lag) at any time.

**FR12:** Cluster failover is automatic; operator intervention is not required for promotion of a healthy replica to primary.

### Backup & Recovery

**FR13:** Operator can perform a manual backup on demand.

**FR14:** Automated backups run daily (at minimum) without operator intervention.

**FR15:** Backups are stored on a separate physical host from the PostgreSQL VMs (no correlated failure).

**FR16:** Operator can restore the cluster from a backup and verify recovery within a ≤ 4 hour RTO.

**FR17:** Operator can verify backup integrity by executing a test restore to an isolated environment.

### Acceptance Testing & Verification

**FR18:** Operator can execute an automated acceptance test: provision a test database, insert test data via automation, verify the data is readable, and tear down the test database cleanly.

**FR19:** Operator can verify that a service role cannot read or write outside its scoped database.

**FR20:** Operator can verify audit log shows all connection events, DDL operations, and privilege escalation attempts.

### Operational Documentation

**FR21:** All provisioning, database creation, credential rotation, backup, restore, and upgrade operations are documented as step-by-step runbooks executable without prior context.

**FR22:** Operator can execute the blast-and-repave procedure end-to-end and verify via the acceptance test.

**FR23:** Operator can document and execute a PostgreSQL minor version upgrade with no manual VM changes.

---

## Non-Functional Requirements

**NFR1: Availability**
Target: 99.95% (homelab HA cluster; planned maintenance windows acceptable; no on-call SLA required). Unplanned downtime must be mitigated by HA topology, not tolerance.

**NFR2: RPO (Recovery Point Objective)**
Maximum acceptable data loss: 24 hours (aligned to daily backup cadence).

**NFR3: RTO (Recovery Time Objective)**
≤ 4 hours from declared incident to verified operational cluster with data confirmed intact.

**NFR4: TLS Enforcement**
All PostgreSQL connections must use TLS. Plaintext connections rejected at the server listener. No exceptions.

**NFR5: No Plaintext Credentials**
Database passwords must never be stored in git, environment files, or any filesystem location outside Vault. All credentials stored in Vault KV v2.

**NFR6: Minimum-Privilege Roles**
No application workload accesses PostgreSQL with superuser privileges. Each service has a dedicated, scoped role. Superuser is a documented break-glass credential stored in Vault.

**NFR7: Blast-and-Repave Reproducibility**
Full cluster reprovisioning from git must be achievable with no manual steps beyond providing the age key.

**Blast-and-Repave Preconditions:** The following must be true before blast-and-repave can proceed:
1. Proxmox is operational and reachable
2. The automation server running OpenTofu is online and functional — this is the execution dependency; if the automation server is lost, it must be restored first
3. The operator age key is in custody
4. Network connectivity exists between the automation server and Proxmox
5. The secrets infrastructure (Vault) is operational — PostgreSQL provisioning depends on Vault for credential storage and retrieval

Blast-and-repave follows dependency order: restore automation server → restore secrets infrastructure → reprovision PostgreSQL cluster.

**NFR8: Pinned Versions**
PostgreSQL version (17.x, pinned at techplan) and all toolchain components must be pinned in OpenTofu lock files. No floating version constraints.

**NFR9: Backup Storage Isolation**
Backups must be stored on a storage path independent of the PostgreSQL VM data volumes — a dedicated backup VM or externally-attached storage volume, not co-resident on the same VM disks as the cluster nodes. A PostgreSQL VM failure (disk corruption, accidental deletion) must not simultaneously destroy backup data. Note: this homelab operates a single physical Proxmox server; physical host total loss is an accepted risk covered under the single-host HA scope deviation (see Compliance section). The isolation requirement targets VM-level failure independence, not physical-host-level independence.

**NFR10: Resource Proportionality**
Cluster sizing must be conservative — the Proxmox cluster hosts many services. VM resource allocation (CPU, RAM, disk) must leave sufficient headroom for k3s and platform services. Projected maximum: ~12 databases, ≤ 100 GB total data.

**NFR11: IaC Authority**
VM provisioning is performed by OpenTofu only. No manual changes to provisioned VMs outside of documented break-glass procedures. Day-2 database management (create database, create role) may use OpenTofu resources, Ansible playbooks, or documented scripts — exact authority is a techplan decision.

**NFR12: Audit Logging**
PostgreSQL structured audit logging captures all connections, privilege escalations, DDL operations, and authentication failures. Logs are JSON-formatted and compatible with future log aggregation pipelines.

**NFR13: HA Architecture Preservation**
The cluster architecture must not make a future move to full HA with PITR or multi-standby more difficult. Avoid lock-in to single-standby-only topologies.

**NFR14: Backup Non-Blocking Guarantee**
Backup operations must never block the PostgreSQL primary. Specifically:
- WAL archiving failures must be bounded — the primary must not stall indefinitely waiting for a failed `archive_command`. The backup tooling selected in TechPlan must support async or non-blocking archive mode, or `archive_command` must be wrapped with an explicit timeout.
- A failed or unreachable backup destination must not halt WAL processing. `max_wal_size` must be tuned to accommodate a reasonable backup-host outage window without risk of disk exhaustion on the primary.
- Backup job failures must produce a detectable log entry, independent of any monitoring solution.

---

## Scope & Phasing

### MVP — Required for k3s Unblock

| Capability | In MVP |
|---|---|
| HA PostgreSQL cluster (primary + replica, auto-failover) | ✅ |
| OpenTofu provisioning of VMs and cluster | ✅ |
| TLS-enforced connections | ✅ |
| Vault-stored credentials at `secret/terminus/{env}/postgres/...` | ✅ |
| Management UI access | ✅ |
| Automated database + service role provisioning | ✅ |
| Daily automated backups to separate host | ✅ |
| Manual backup on demand | ✅ |
| Backup restore procedure + verified RTO measurement | ✅ |
| Blast-and-repave procedure + verification | ✅ |
| Automated acceptance test (test DB, data insert, teardown, audit check) | ✅ |
| All operational runbooks | ✅ |
| SoD deviation record | ✅ |

### Explicitly Out of MVP Scope

| Capability | Reason for Deferral |
|---|---|
| Monitoring / alerting | Dedicated future initiative — monitoring is a cross-cutting concern for the whole ecosystem, not owned by postgres. Postgres must be *compatible* (structured logs, Prometheus-ready metrics endpoint) but does not deliver the solution. |
| PITR (point-in-time recovery) | Growth phase — daily backups satisfy RPO for MVP |
| Connection pooling (pgBouncer) | Deferred until consumer load is understood |
| Automated credential rotation | Growth phase — manual rotation with runbook is MVP |
| Schema migration tooling (Liquibase) | Each consuming service owns its own migration strategy; centralized tooling is growth |
| Automated failover testing on a schedule | Growth phase — manual verification at MVP |
| Metrics export (Prometheus) | Growth phase — tied to monitoring solution |

### Growth Phase Candidates

- Prometheus metrics export (when monitoring solution is defined)
- PITR with WAL archiving
- Connection pooling via pgBouncer
- Automated credential rotation (Vault dynamic secrets — postgres secrets engine)
- Additional replica nodes / geo-distribution
- Automated scheduled DR tests

---

## Architecture Constraints & Decisions (Input to TechPlan)

| Decision | Direction | Status |
|---|---|---|
| Deployment target | Dedicated Proxmox VMs (not in k3s) | **Decided** |
| Provisioning authority | OpenTofu for VM and cluster provisioning | **Decided** |
| Credential storage | Vault KV v2 at `secret/terminus/{env}/postgres/{dbname}/...` | **Decided** |
| PostgreSQL version | 17.x (pin specific release at techplan) | **Decided** |
| HA topology | Primary + replica, automatic failover; both VMs on single physical host — see HA Scope Limitation deviation | **Decided** |
| TLS | Required, self-signed CA from infra TLS CA (aligns with secrets D6) | **Decided** |
| Authentication model | Password-over-TLS using Vault KV v2 static credentials; TLS client cert auth deferred — avoids PKI complexity beyond current secrets foundation; use Vault to its fullest within established scope | **Decided** |
| Backup location | Isolated from PostgreSQL VM data volumes; dedicated backup VM or externally-attached storage; single physical host means physical host loss is accepted risk — see HA Scope Limitation deviation | **Decided** |
| State backend | OpenTofu local state, SOPS-encrypted before commit (aligns with secrets D4) | **Decided** |
| HA tooling | Patroni vs. repmgr vs. other | **Open — TechPlan** |
| Database/role provisioning tooling | OpenTofu resource vs. Ansible playbook vs. script | **Open — TechPlan** |
| Backup tooling | pg_dump + cron vs. pgBackRest vs. Barman | **Open — TechPlan** |
| Connection model | Direct TCP vs. pgBouncer vs. other | **Open — TechPlan** |
| Schema migration ownership | Each consumer vs. centralized | **Open — TechPlan** |
| VM sizing | CPU/RAM/disk allocation | **Open — TechPlan** |

---

## Compliance & Deviations

### Fintech Domain — Applicability Statement

This initiative is classified under the **fintech** domain for security posture purposes (skills-transfer goal: patterns must be production-transferable to fintech environments). Standard fintech regulatory requirements (KYC/AML, PCI DSS, open banking APIs, crypto regulations) are **not applicable** to a homelab infrastructure project with no payment processing, no customer data, and no regulatory jurisdiction.

The fintech classification drives the following binding requirements:

| Requirement Class | Status |
|---|---|
| Security-first posture | ✅ Required — NFR4, NFR5, NFR6, NFR12 |
| Audit logging | ✅ Required — FR20, NFR12 |
| Credential lifecycle management | ✅ Required — FR3, FR4, FR6, FR9 |
| Change control via PR workflow | ✅ Required — NFR11 |
| KYC / AML | ❌ Not applicable — no customer identity or transaction data |
| PCI DSS | ❌ Not applicable — no payment card data |
| Fraud prevention controls | ❌ Not applicable — internal homelab only |
| External regulatory reporting | ❌ Not applicable |

**Fraud Prevention:** Not applicable. No financial transactions or external users. Sole operator environment.

---

### Separation of Duties Deviation

**Context:** This is a homelab operated by a single person (CrisWeber). All roles — DBA, infrastructure operator, security officer — are held by the same individual.

**Risk:** An error or compromise affecting the operator simultaneously affects all administrative functions.

**Compensating Controls:**
- All changes via PR workflow (self-review is documented as accepted deviation)
- Audit logging captures all privileged operations
- Blast-and-repave and backup restore are the primary recovery mechanisms
- Break-glass usage triggers mandatory rotation of all impacted credentials

**Status:** Risk accepted. Formal deviation record to be committed as `docs/terminus/infra/postgres/deviations/sod-deviation.md` during techplan.

---

### Single Physical Host — HA Scope Limitation

**Context:** This homelab operates a single physical Proxmox server. Both the PostgreSQL primary and replica VMs run on the same physical host.

**Scope of HA Guarantee:** The HA cluster protects against VM-level failures — OS crashes, PostgreSQL process failures, VM corruption, accidental VM deletion. It does **not** protect against physical host failure. A complete physical host loss results in total cluster loss, requiring full reprovision and restore from backup (→ Journey 4, FR16).

**Purpose of HA in This Context:**
- **Skill development:** operating, managing, and troubleshooting a real HA PostgreSQL cluster is a primary goal; patterns produced here are directly transferable to multi-host production environments
- **Patch safety:** a cluster node can be taken offline for patching while the cluster remains operational on the surviving VM, reducing change-window risk
- **Practice:** failover, replication, and recovery procedures are exercised in a realistic topology

**Risk:** Physical host failure causes total cluster loss — both VMs lost simultaneously.

**Compensating Control:** Daily automated backups to isolated storage (NFR9) provide the recovery path. Total host loss → restore from backup + full reprovision via blast-and-repave (Journey 4).

**Status:** Risk accepted. No physical-host-level mitigation available given single-server constraint.

---

## Open Questions

| # | Question | Owner | Target Resolution |
|---|---|---|---|
| OQ1 | HA tooling selection: Patroni vs. repmgr vs. pg_auto_failover | Todd | ✅ Resolved — TechPlan: **Patroni + Consul DCS** |
| OQ2 | Database / role provisioning mechanism: OpenTofu resource, Ansible playbook, or script? | Todd | ✅ Resolved — TechPlan: **OpenTofu `cyrilgdn/postgresql` + `hashicorp/vault` providers** |
| OQ3 | Backup tooling: pg_dump + cron, pgBackRest, or Barman? | Todd | ✅ Resolved — TechPlan: **pgBackRest, async WAL, full weekly + diff daily** |
| OQ4 | Connection model: direct TCP or connection pooler? | Todd | ✅ Resolved — TechPlan: **Direct TCP via `master.postgresql.service.consul`** |
| OQ5 | VM sizing: CPU, RAM, disk per node? | Todd | ✅ Resolved — TechPlan: **4 vCPU / 16 GB / 182 GB × 2 nodes; backup VM 2 vCPU / 4 GB / 300 GB** |
| OQ6 | How many cluster nodes? Primary + 1 replica, or primary + 2 replicas? | Todd | ✅ Resolved — TechPlan: **Primary + 1 replica (2 nodes; single-host deviation)** |
| OQ7 | Monitoring/alerting solution — dedicated future initiative; postgres delivers structured logs (NFR12) and a Prometheus-compatible metrics endpoint (growth phase) as integration points; postgres must not be designed to preclude this | Todd | Future initiative |

---

## Required MVP Runbooks

All eight runbooks are first-class deliverables, not optional documentation:

- `docs/runbooks/postgres-day0-bootstrap.md`
- `docs/runbooks/postgres-blast-and-repave.md`
- `docs/runbooks/postgres-backup.md`
- `docs/runbooks/postgres-restore.md`
- `docs/runbooks/postgres-new-database.md`
- `docs/runbooks/postgres-credential-rotation.md`
- `docs/runbooks/postgres-upgrade.md`
- `docs/runbooks/postgres-break-glass.md`

---

## FR Coverage Map

| FR | Success Criteria | Journey | Section |
|---|---|---|---|
| FR1, FR2 | SC1, SC5 | J1, J4 | Provisioning |
| FR3, FR4, FR5 | SC2 | J2 | Provisioning + Credential Delivery |
| FR6, FR8, FR9 | SC2 | J2 | Access & Auth |
| FR7 | SC1 | J1, J3 | Access & Auth |
| FR10, FR11, FR12 | SC5 | J4 | High Availability |
| FR13, FR14, FR15 | SC4 | J3, J4 | Backup & Recovery |
| FR16, FR17 | SC5 | J4 | Backup & Recovery |
| FR18, FR19, FR20 | SC3, SC5 | J1, J2, J4 | Acceptance Testing |
| FR21, FR22, FR23 | SC5 | J3, J4 | Operational Documentation |
