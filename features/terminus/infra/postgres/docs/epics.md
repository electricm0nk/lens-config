---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/infra/postgres/prd.md
  - docs/terminus/infra/postgres/architecture.md
---

# terminus-infra-postgres — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the `terminus-infra-postgres` initiative, decomposing requirements from the PRD and Architecture into implementable stories.

---

## Requirements Inventory

### Functional Requirements

FR1: Operator can provision a complete, operational HA PostgreSQL cluster from zero via a single `tofu apply` with no manual VM steps.
FR2: Operator can reprovision the cluster entirely from git after full VM destruction (blast-and-repave) with no manual steps beyond providing the age key.
FR3: Operator can provision a new PostgreSQL database with a dedicated service role via automation; the process requires no direct VM access.
FR4: Newly provisioned database credentials are automatically stored in Vault KV v2 at `secret/terminus/{env}/postgres/{dbname}` with schema: username, password, host, port, db_name, ssl_mode. All fields always populated.
FR5: Operator can retrieve database credentials from Vault KV v2 using the established secrets infrastructure.
FR6: Each database has a dedicated service role scoped to that database only; no application workload uses a superuser account.
FR7: Operator can access pgAdmin 4 on their workstation and connect to the cluster over TLS for monitoring, verification, and authorized ad-hoc queries.
FR8: PostgreSQL accepts TLS connections only; plaintext connections are rejected at the listener.
FR9: Operator can revoke a service role credential and issue a new one without affecting other database identities.
FR10: PostgreSQL cluster provides HA such that a single node failure causes no data loss and no service interruption requiring operator intervention.
FR11: Operator can verify cluster HA status (primary, replica health, replication lag) at any time.
FR12: Cluster failover is automatic; operator intervention is not required for promotion of a healthy replica to primary.
FR13: Operator can perform a manual backup on demand.
FR14: Automated backups run daily (at minimum) without operator intervention.
FR15: Backups are stored on a separate VM from the PostgreSQL cluster VMs (no correlated failure).
FR16: Operator can restore the cluster from a backup and verify recovery within a ≤ 4 hour RTO.
FR17: Operator can verify backup integrity by executing a test restore to an isolated environment.
FR18: Operator can execute an automated acceptance test: provision a test database, insert test data, verify data is readable, and tear down the test database cleanly.
FR19: Operator can verify that a service role cannot read or write outside its scoped database.
FR20: Operator can verify audit log shows all connection events, DDL operations, and privilege escalation attempts.
FR21: All provisioning, database creation, credential rotation, backup, restore, and upgrade operations are documented as step-by-step runbooks executable without prior context.
FR22: Operator can execute the blast-and-repave procedure end-to-end and verify via the acceptance test.
FR23: Operator can document and execute a PostgreSQL minor version upgrade with no manual VM changes.

### Non-Functional Requirements

NFR1: Availability — 99.95% target; HA topology mitigates unplanned downtime; VM-level failure scope.
NFR2: RPO — Maximum acceptable data loss: 24 hours (aligned to daily backup cadence).
NFR3: RTO — ≤ 4 hours from declared incident to verified operational cluster with data confirmed intact.
NFR4: TLS Enforcement — All PostgreSQL connections must use TLS; plaintext connections rejected at the listener.
NFR5: No Plaintext Credentials — Database passwords never stored in git, env files, or any filesystem outside Vault.
NFR6: Minimum-Privilege Roles — No application workload uses superuser; each service has a dedicated scoped role.
NFR7: Blast-and-Repave Reproducibility — Full cluster reprovisioning from git with no manual steps beyond age key. Bootstrap order: Proxmox → automation server → Vault → PostgreSQL.
NFR8: Pinned Versions — PostgreSQL 17.x and all toolchain components pinned in OpenTofu lock files.
NFR9: Backup Storage Isolation — Backups on a separate VM from cluster nodes; VM-level failure independence.
NFR10: Resource Proportionality — Cluster footprint ≤ ~13% host RAM; leaves headroom for k3s and platform services.
NFR11: IaC Authority — VM provisioning by OpenTofu only; no manual VM changes outside documented break-glass procedures.
NFR12: Audit Logging — Structured JSON audit logs (pgaudit) capturing connections, DDL, privilege escalations; compatible with future log aggregation.
NFR13: HA Architecture Preservation — Architecture must not preclude future PITR, multi-standby, or pgBouncer insertion.
NFR14: Backup Non-Blocking Guarantee — `archive_command` must be async (`archive-async=y`); backup failures produce detectable log entries; primary must not stall on backup VM outage.

### Additional Requirements (from Architecture)

- **State management:** SOPS-local state (`terraform.tfstate.enc`) encrypted with operator age key before git commit — no remote state backend for this capability
- **HA tooling:** Patroni + Consul DCS; Consul 1.22.5 already in stack
- **VM sizing:** postgres-primary-vm and postgres-replica-vm: 4 vCPU / 16 GB RAM / 32 GB OS + 150 GB data; postgres-backup-vm: 2 vCPU / 4 GB / 300 GB
- **Backup tooling:** pgBackRest; `archive-async=y`; `archive-push-queue-max=4GB`; `max_wal_size=2GB`; full weekly (Sunday 02:00) + diff daily (Mon-Sat 02:00); 2-week retention
- **Provisioning order dependency:** `tofu apply` (all modules including `postgres-backup`) must complete before pgBackRest `remote-exec` provisioners run on cluster nodes; stanza-create runs after backup VM IP is known (module output) and cluster node config is complete
- **TLS:** Server cert from infra TLS CA; `ssl=on`; `pg_hba.conf` hostssl-only entries
- **Audit:** `pgaudit` extension; `log_destination=jsonlog`; 30-day rotation via logrotate
- **Vault paths:** `secret/terminus/{env}/postgres/{dbname}` (credential), `secret/terminus/{env}/postgres/admin` (break-glass), `secret/terminus/{env}/postgres/replication`, `secret/terminus/{env}/postgres/pgbackrest-ssh-key`, `secret/terminus/{env}/postgres/patroni-consul-token`
- **Naming conventions:** VMs `postgres-{role}-vm`; Patroni scope `postgres-{env}`; pgBackRest stanza `main-{env}`; Consul service `master.postgresql.service.consul`
- **3 ADRs to create:** ADR-0002 (Patroni+Consul), ADR-0003 (pgBackRest), ADR-0004 (OpenTofu providers)
- **SoD deviation doc:** `docs/terminus/infra/postgres/deviations/sod-deviation.md` (deferred from TechPlan)
- **Demo database:** One functional demo/acceptance database (`acceptance_test_db`) delivered and tested as part of this initiative; named consuming-service databases provisioned by their own initiatives

### FR Coverage Map

| FR | Epic | Description |
|---|---|---|
| FR1 | EPIC-001 | Provision cluster VMs via `tofu apply` |
| FR2 | EPIC-001 | Blast-and-repave reprovisioning from git |
| FR3 | EPIC-004 | Provision database + role via automation |
| FR4 | EPIC-004 | Credentials auto-written to Vault KV v2 |
| FR5 | EPIC-004 | Retrieve credentials from Vault |
| FR6 | EPIC-004 | Per-database scoped role, no superuser for apps |
| FR7 | EPIC-002 | pgAdmin4 workstation connects to cluster over TLS |
| FR8 | EPIC-002 | TLS-only listener, plaintext rejected |
| FR9 | EPIC-004 | Credential revocation and reissue |
| FR10 | EPIC-002 | Single node failure → no data loss, no intervention |
| FR11 | EPIC-002 | Cluster HA status verifiable at any time |
| FR12 | EPIC-002 | Automatic failover; no operator intervention |
| FR13 | EPIC-003 | Manual on-demand backup |
| FR14 | EPIC-003 | Daily automated backups |
| FR15 | EPIC-003 | Backup VM isolated from cluster VMs |
| FR16 | EPIC-004 | Restore from backup + RTO ≤ 4h (acceptance test) |
| FR17 | EPIC-003 | Test restore to isolated path verifies integrity |
| FR18 | EPIC-004 | Automated acceptance test: provision → insert → verify → teardown |
| FR19 | EPIC-004 | Role isolation: cannot access other databases |
| FR20 | EPIC-004 | Audit log captures connections, DDL, privilege escalations |
| FR21 | EPIC-005 | All 8 runbooks exist and are executable |
| FR22 | EPIC-005 | Blast-and-repave procedure executable end-to-end |
| FR23 | EPIC-005 | Minor version upgrade procedure documented and executable |

---

## Epic List

1. **EPIC-001** — Infrastructure Provisioning
2. **EPIC-002** — Cluster Bootstrap & High Availability
3. **EPIC-003** — Backup Infrastructure
4. **EPIC-004** — Database Provisioning & Acceptance Testing
5. **EPIC-005** — Operational Runbooks

---

## EPIC-001: Infrastructure Provisioning

**Operator outcome:** Todd can run `tofu apply` and get three provisioned, correctly-sized Proxmox VMs (primary, replica, backup) with SOPS-encrypted state committed to git. Three ADRs documenting key architecture decisions are also committed. No PostgreSQL cluster yet — standing infrastructure ready for bootstrap.

**FRs covered:** FR1, FR2 (partial — VMs provisionable; full blast-and-repave validated in EPIC-005)
**NFRs covered:** NFR8, NFR10, NFR11

---

## EPIC-002: Cluster Bootstrap & High Availability

**Operator outcome:** Todd can connect to a live, TLS-enforced HA PostgreSQL 17 cluster via pgAdmin4 on his workstation, verify cluster health via `patronictl list`, trigger a deliberate node failure or switchover, and watch Patroni automatically promote the replica — all without operator intervention. Audit logging is active and producing structured JSON output.

**FRs covered:** FR7, FR8, FR10, FR11, FR12
**NFRs covered:** NFR1, NFR4, NFR12, NFR13

---

## EPIC-003: Backup Infrastructure

**Operator outcome:** Todd can run an on-demand `pgbackrest` backup, confirm the daily automated schedule is active, verify that WAL archiving is asynchronous and non-blocking, and execute a test restore to an isolated path that confirms data integrity. The backup VM is fully configured with independent storage.

**FRs covered:** FR13, FR14, FR15, FR17
**NFRs covered:** NFR2, NFR3, NFR9, NFR14

---

## EPIC-004: Database Provisioning & Acceptance Testing

**Operator outcome:** Todd can declare a database in OpenTofu, run `tofu apply`, retrieve the full credential bundle from Vault, connect as the scoped role, verify it cannot access other databases, rotate credentials and confirm the old ones are revoked, and run all 5 automated acceptance tests — covering cluster health, database lifecycle, role isolation, credential revocation, audit logging, and backup/restore verification — all passing.

**FRs covered:** FR3, FR4, FR5, FR6, FR9, FR16, FR18, FR19, FR20
**NFRs covered:** NFR5, NFR6

---

## EPIC-005: Operational Runbooks

**Operator outcome:** Todd can execute any cluster maintenance procedure — bootstrapping, blast-and-repave, backup, restore, new database creation, credential rotation, version upgrade, break-glass — from the runbook alone, without relying on memory or prior context. SoD deviation doc is committed. FR22 (blast-and-repave E2E) and FR23 (minor version upgrade) are both documented and have been walked through at least once.

**FRs covered:** FR21, FR22, FR23
**NFRs covered:** NFR3, NFR7

---

## Stories

### EPIC-001: Infrastructure Provisioning

---

#### Story 1.1 — Provision PostgreSQL cluster VMs via OpenTofu

**As an** operator, **I want to** provision the primary and replica PostgreSQL VMs by running `tofu apply`, **so that** I have correctly-sized, network-reachable Proxmox VMs ready for cluster bootstrap.

**Acceptance Criteria:**
- [ ] 2 VMs exist in Proxmox with correct specs: 4 vCPU / 16 GB RAM / 32 GB OS + 150 GB data disk
- [ ] Both VMs are reachable over SSH from the automation server
- [ ] Hostnames resolve in the Terminus network
- [ ] All provider versions are pinned and locked in `.terraform.lock.hcl`

---

#### Story 1.2 — Provision PostgreSQL backup VM via OpenTofu

**As an** operator, **I want to** provision the backup VM by running `tofu apply`, **so that** I have an isolated, correctly-sized storage host for pgBackRest before backup configuration begins.

**Acceptance Criteria:**
- [ ] `postgres-backup-vm` exists in Proxmox: 2 vCPU / 4 GB RAM / 300 GB storage
- [ ] VM is reachable over SSH from both cluster nodes
- [ ] 300 GB volume is mounted at `/var/lib/pgbackrest`
- [ ] Module outputs backup VM IP (consumed by `remote-exec` provisioners in EPIC-003)

---

#### Story 1.3 — SOPS-encrypted local state committed to git

**As an** operator, **I want** OpenTofu state encrypted with my age key and committed to git, **so that** infrastructure state is version-controlled, never plaintext, and recoverable from git without a remote state backend.

**Acceptance Criteria:**
- [ ] `terraform.tfstate.enc` is committed to git
- [ ] Plaintext `terraform.tfstate` is never present in git at any point
- [ ] `sops -d terraform.tfstate.enc > terraform.tfstate` succeeds before subsequent `tofu plan`
- [ ] `.gitignore` includes `terraform.tfstate`

---

#### Story 1.4 — Commit infrastructure ADRs

**As an** operator, **I want** the three architectural decisions recorded as ADRs, **so that** future maintainers understand why Patroni, pgBackRest, and the OpenTofu provider pattern were chosen.

**Acceptance Criteria:**
- [ ] ADR-0002 (Patroni + Consul DCS) exists in `terminus.infra/docs/decisions/`
- [ ] ADR-0003 (pgBackRest) exists in `terminus.infra/docs/decisions/`
- [ ] ADR-0004 (OpenTofu cyrilgdn/postgresql + hashicorp/vault providers) exists in `terminus.infra/docs/decisions/`
- [ ] Each ADR follows the ADR-0001 format (context, decision, consequences)

---

### EPIC-002: Cluster Bootstrap & High Availability

---

#### Story 2.1 — Install PostgreSQL 17 + Patroni via OpenTofu remote-exec

**As an** operator, **I want** PostgreSQL 17 and Patroni installed on both cluster VMs via OpenTofu `remote-exec` provisioners, **so that** the cluster nodes have all required packages before Patroni cluster initialization.

**Acceptance Criteria:**
- [ ] PostgreSQL 17.x installed on both VMs, version pinned in `remote-exec` script
- [ ] Patroni installed at pinned version on both VMs
- [ ] `postgresql.service` disabled on both VMs (Patroni owns process lifecycle)
- [ ] Prerequisite packages (python3-psycopg2, etc.) installed

---

#### Story 2.2 — Bootstrap Patroni cluster with Consul DCS

**As an** operator, **I want** Patroni initialized with Consul as the DCS and both nodes joined to the cluster, **so that** I have a functioning primary + replica pair with automatic leader election.

**Acceptance Criteria:**
- [ ] `patronictl -c /etc/patroni/patroni.yml list` shows one Leader and one Replica, both healthy
- [ ] Patroni scope is `postgres-{env}`
- [ ] Consul service `master.postgresql.service.consul` resolves to primary
- [ ] `patroni-consul-token` written to `secret/terminus/{env}/postgres/patroni-consul-token` in Vault

---

#### Story 2.3 — Enforce TLS-only connections

**As an** operator, **I want** PostgreSQL to accept only TLS connections and reject plaintext, **so that** NFR4 is satisfied and all traffic is encrypted at the listener.

**Acceptance Criteria:**
- [ ] `ssl = on` in `postgresql.conf`
- [ ] `pg_hba.conf` contains only `hostssl` entries; no plaintext `host` entries exist
- [ ] `psql --no-ssl` connection attempt is rejected by the server
- [ ] Server cert is sourced from infra TLS CA

---

#### Story 2.4 — Configure pgaudit + structured JSON logging

**As an** operator, **I want** pgaudit producing structured JSON logs capturing connections, DDL, and privilege escalations, **so that** NFR12 is satisfied and audit evidence is available.

**Acceptance Criteria:**
- [ ] `pgaudit` extension loaded
- [ ] `log_destination = jsonlog` in `postgresql.conf`
- [ ] Connection event appears in JSON log after test connection
- [ ] DDL event (CREATE TABLE) appears in JSON log
- [ ] Log rotation configured via logrotate with 30-day retention

---

#### Story 2.5 — Verify automatic HA failover via deliberate switchover

**As an** operator, **I want to** trigger a deliberate `patronictl switchover` and confirm automatic promotion, **so that** FR10, FR11, and FR12 are verified and I have confidence in the HA topology.

**Acceptance Criteria:**
- [ ] `patronictl switchover` completes without operator intervention
- [ ] New primary is healthy and accepting connections within 30 seconds
- [ ] `master.postgresql.service.consul` resolves to new primary after switchover
- [ ] No data loss confirmed (row inserted before switchover is readable after)

---

#### Story 2.6 — Verify pgAdmin4 workstation connection over TLS

**As an** operator, **I want to** open pgAdmin4 on my workstation and connect to `master.postgresql.service.consul:5432` over TLS, **so that** FR7 is verified and I have a working management interface.

**Acceptance Criteria:**
- [ ] pgAdmin4 connects using `sslmode=require`
- [ ] Connection uses break-glass admin credential retrieved from Vault (`secret/terminus/{env}/postgres/admin`)
- [ ] pgAdmin4 shows database list and reports server version 17.x
- [ ] Plaintext connection attempt (`sslmode=disable`) from pgAdmin4 is rejected

---

### EPIC-003: Backup Infrastructure

---

#### Story 3.1 — Install and configure pgBackRest on all nodes

**As an** operator, **I want** pgBackRest installed on all three VMs and `pgbackrest.conf` written on each via `remote-exec`, **so that** WAL archiving and backup jobs can route to the backup VM repository.

**Acceptance Criteria:**
- [ ] pgBackRest installed on `postgres-primary-vm`, `postgres-replica-vm`, and `postgres-backup-vm`
- [ ] `/etc/pgbackrest/pgbackrest.conf` written on each VM by `remote-exec` provisioner; backup VM IP sourced from `postgres-backup` module output
- [ ] SSH key pair generated and stored in Vault (`secret/terminus/{env}/postgres/pgbackrest-ssh-key`); public key deployed to backup VM
- [ ] `pgbackrest --stanza=main-{env} check` passes on primary node

---

#### Story 3.2 — Initialize pgBackRest stanza

**As an** operator, **I want** the pgBackRest stanza `main-{env}` created on the backup VM, **so that** the backup repository is initialized and ready to accept WAL archives and backups.

**Acceptance Criteria:**
- [ ] `pgbackrest --stanza=main-{env} stanza-create` completes successfully on backup VM (run via `remote-exec` after cluster node config in Story 3.1)
- [ ] Repository directory structure exists at `/var/lib/pgbackrest`
- [ ] `pgbackrest --stanza=main-{env} info` returns stanza status without error

---

#### Story 3.3 — Configure async WAL archiving

**As an** operator, **I want** WAL archiving enabled with `archive-async=y`, **so that** the primary is never blocked by backup VM outage (NFR14).

**Acceptance Criteria:**
- [ ] `archive_mode = on` and `archive_command` set in `postgresql.conf`
- [ ] `archive-async = y` and `archive-push-queue-max = 4GB` in `pgbackrest.conf`
- [ ] WAL segment appears in backup repository after test transaction
- [ ] Simulated backup VM pause does not cause primary to stall (confirmed via log inspection)

---

#### Story 3.4 — Configure automated backup schedule

**As an** operator, **I want** full weekly and differential daily backups running automatically without operator intervention, **so that** FR14 is satisfied and RPO ≤ 24h (NFR2) is met.

**Acceptance Criteria:**
- [ ] Cron job on backup VM runs full backup every Sunday 02:00 (`pgbackrest --stanza=main-{env} backup --type=full`)
- [ ] Cron job on backup VM runs diff backup Mon–Sat 02:00 (`--type=diff`)
- [ ] `pgbackrest --stanza=main-{env} info` shows at least one completed backup after manual trigger
- [ ] 2-week retention policy configured (`repo1-retention-full=2`)

---

#### Story 3.5 — On-demand backup and test restore to isolated path

**As an** operator, **I want to** run a manual backup on demand and restore it to an isolated path to verify integrity, **so that** FR13 and FR17 are satisfied and backup tooling is confirmed end-to-end before EPIC-004 acceptance tests.

**Acceptance Criteria:**
- [ ] `pgbackrest --stanza=main-{env} backup --type=full` completes successfully on demand
- [ ] Test restore to isolated path (`--pg1-path=/tmp/pgbackrest-restore-test`) succeeds
- [ ] Restored data directory contains expected PostgreSQL cluster files
- [ ] Original cluster is unaffected by the test restore

---

### EPIC-004: Database Provisioning & Acceptance Testing

---

#### Story 4.1 — Implement `postgres-databases` OpenTofu module

**As an** operator, **I want** a reusable `postgres-databases` OpenTofu module that provisions a database, scoped role, grants, and Vault credential write in a single `tofu apply`, **so that** FR3–FR6 are covered by a repeatable, idempotent pattern.

**Acceptance Criteria:**
- [ ] Module accepts `db_name` and `environment` as inputs
- [ ] Creates `postgresql_database`, `postgresql_role` (login, no superuser), `postgresql_grant` (CONNECT + CREATE on own DB only)
- [ ] Writes full credential bundle to `secret/terminus/{env}/postgres/{dbname}` in Vault KV v2 (username, password, host, port, db_name, ssl_mode)
- [ ] `tofu apply` is idempotent — re-run produces no changes if state matches

---

#### Story 4.2 — Provision `acceptance_test_db` and verify Vault credentials

**As an** operator, **I want to** provision the demo database `acceptance_test_db` and retrieve its credentials from Vault, **so that** FR3, FR4, and FR5 are exercised end-to-end.

**Acceptance Criteria:**
- [ ] `acceptance_test_db` database exists in the cluster
- [ ] `acceptance_test_db_app` role exists with login privilege and no superuser rights
- [ ] Full credential bundle readable from `secret/terminus/{env}/postgres/acceptance_test_db`
- [ ] `psql` connection using Vault credentials succeeds with `sslmode=require`

---

#### Story 4.3 — Verify role isolation (FR6, FR19)

**As an** operator, **I want to** confirm that `acceptance_test_db_app` cannot read or write to any database other than `acceptance_test_db`, **so that** FR6 and FR19 are verified.

**Acceptance Criteria:**
- [ ] Connect as `acceptance_test_db_app`; attempt to connect to `postgres` database is denied
- [ ] Attempt to CREATE TABLE in the `postgres` database is denied
- [ ] SELECT on a table in another database is denied
- [ ] All denials are confirmed in pgaudit JSON log

---

#### Story 4.4 — Credential revocation and reissue (FR9)

**As an** operator, **I want to** revoke the `acceptance_test_db_app` credential and issue a new one without affecting other database identities, **so that** FR9 is verified (acceptance Test 4).

**Acceptance Criteria:**
- [ ] Old credential (`psql` connect) works before rotation
- [ ] `tofu apply` with credential rotation writes new credential to Vault
- [ ] Old credential is rejected by PostgreSQL after rotation (`FATAL: password authentication failed`)
- [ ] New credential retrieved from Vault connects successfully
- [ ] Other database credentials are unaffected

---

#### Story 4.5 — Run acceptance Tests 1–3 (cluster health, DB lifecycle, role isolation)

**As an** operator, **I want** the first three automated acceptance tests to pass, **so that** cluster health, database lifecycle, and role isolation are formally verified.

**Acceptance Criteria:**
- [ ] **Test 1 (Cluster Health):** `patronictl list` shows Leader + Replica healthy; replication lag = 0
- [ ] **Test 2 (DB Lifecycle):** Provision test DB → insert row → verify row readable → drop DB → verify gone; all via OpenTofu + psycopg2 script
- [ ] **Test 3 (Role Isolation):** `acceptance_test_db_app` denied access to all other databases (automated script, mirrors Story 4.3)

---

#### Story 4.6 — Run acceptance Tests 5–6 (audit log, backup/restore verification)

**As an** operator, **I want** the final two automated acceptance tests to pass, **so that** audit logging and backup/restore RTO ≤ 4h are formally verified.

**Acceptance Criteria:**
- [ ] **Test 5 (Audit Log):** Connection event, DDL (CREATE TABLE), and privilege escalation attempt all appear in JSON audit log
- [ ] **Test 6 (Backup/Restore Verification):** Full backup taken → cluster VMs destroyed → `tofu apply` reprovisioning → restore from backup → `acceptance_test_db` data intact; total elapsed time ≤ 4h (NFR3, FR16)

---

### EPIC-005: Operational Runbooks

---

#### Story 5.1 — Runbook: Initial cluster provisioning

**As an** operator, **I want** a runbook covering end-to-end initial provisioning from zero, **so that** the cluster can be stood up by following steps alone without prior context.

**Acceptance Criteria:**
- [ ] Runbook covers: prerequisites check → SOPS key setup → `tofu apply` → bootstrap verification → Patroni health check
- [ ] Every command is copy-pasteable with no placeholders requiring prior context
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/01-initial-provisioning.md`

---

#### Story 5.2 — Runbook: Blast-and-repave (FR22)

**As an** operator, **I want** a runbook to reprovision the cluster from full VM destruction using only git and an age key, **so that** FR22 is satisfied and the procedure has been walked through at least once.

**Acceptance Criteria:**
- [ ] Runbook covers: VM deletion → `sops -d` state decrypt → `tofu apply` reprovisioning → pgBackRest restore → acceptance test verification
- [ ] Procedure executed end-to-end at least once; outcome recorded in runbook
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/02-blast-and-repave.md`

---

#### Story 5.3 — Runbook: Manual backup and restore

**As an** operator, **I want** a runbook for on-demand backup, restore, and RTO verification, **so that** any operator can execute a recovery without prior context.

**Acceptance Criteria:**
- [ ] Covers: manual backup trigger → backup verification → full restore procedure → data integrity check
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/03-backup-and-restore.md`

---

#### Story 5.4 — Runbook: New database provisioning

**As an** operator, **I want** a runbook for provisioning a new database and retrieving its credentials from Vault, **so that** consuming teams can onboard a new database without involving the original implementors.

**Acceptance Criteria:**
- [ ] Covers: add database declaration to OpenTofu → `tofu apply` → retrieve credentials via `vault kv get` → verify connection
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/04-new-database.md`

---

#### Story 5.5 — Runbook: Credential rotation

**As an** operator, **I want** a runbook for revoking and reissuing a service role credential, **so that** FR9 can be executed on demand without prior context.

**Acceptance Criteria:**
- [ ] Covers: trigger rotation in OpenTofu → `tofu apply` → verify old credential rejected → verify new credential works
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/05-credential-rotation.md`

---

#### Story 5.6 — Runbook: HA failover and recovery

**As an** operator, **I want** a runbook for manual switchover, unplanned failover response, and re-joining a recovered node, **so that** any HA event can be handled without prior context.

**Acceptance Criteria:**
- [ ] Covers: `patronictl switchover` → health verification → re-joining replica after repair
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/06-ha-failover.md`

---

#### Story 5.7 — Runbook: Minor version upgrade (FR23)

**As an** operator, **I want** a runbook for upgrading the PostgreSQL minor version with no manual VM changes, **so that** FR23 is satisfied and the procedure has been walked through at least once.

**Acceptance Criteria:**
- [ ] Covers: update version pin in OpenTofu script → `tofu apply` (`remote-exec` re-runs package upgrade) → verify `SELECT version()` reports new version
- [ ] Procedure validated at least once; outcome recorded in runbook
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/07-minor-version-upgrade.md`

---

#### Story 5.8 — Runbook: Break-glass access

**As an** operator, **I want** a runbook for retrieving the break-glass admin credential and accessing the cluster directly, **so that** emergency access is always recoverable from documentation alone.

**Acceptance Criteria:**
- [ ] Covers: `vault kv get secret/terminus/{env}/postgres/admin` → direct `psql` connection → post-use steps (audit, rotate)
- [ ] Stored at `docs/terminus/infra/postgres/runbooks/08-break-glass.md`

---

#### Story 5.9 — SoD deviation doc

**As an** operator, **I want** the single-operator SoD deviation formally documented, **so that** the A6 adversarial finding is closed and future auditors have a record of the deliberate deviation.

**Acceptance Criteria:**
- [ ] `docs/terminus/infra/postgres/deviations/sod-deviation.md` created
- [ ] Documents: what SoD requires, what the homelab reality is, who approved the deviation, and mitigating controls in place

