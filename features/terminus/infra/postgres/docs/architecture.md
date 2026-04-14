---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/terminus/infra/postgres/prd.md
  - docs/terminus/infra/postgres/businessplan-questions.md
  - TargetProjects/lens/lens-governance/constitutions/org/constitution.md
  - TargetProjects/lens/lens-governance/constitutions/terminus/constitution.md
  - TargetProjects/lens/lens-governance/constitutions/terminus/infra/constitution.md
  - TargetProjects/terminus/infra/terminus.infra/docs/decisions/adr-0001-proxmox-state-and-vault-conventions.md
workflowType: architecture
initiative: terminus-infra-postgres
phase: techplan
date: "2026-03-25"
author: Todd
status: complete
completedAt: "2026-03-25"
---

# Architecture Decision Document — terminus-infra-postgres

**Author:** Todd
**Date:** 2026-03-25
**Initiative:** terminus-infra-postgres
**Track:** feature
**Phase:** TechPlan
**Target Repo:** terminus.infra

---

## Executive Summary

This document defines the complete technical architecture for the `terminus-infra-postgres` initiative — a highly available PostgreSQL cluster deployed on dedicated Proxmox VMs as shared infrastructure for the Terminus homelab.

All seven open questions from the PRD are resolved here. The architecture is built entirely on components already present in the Terminus stack (OpenTofu, Vault KV v2, Consul) — no new infrastructure categories are introduced. The result is a two-node Patroni cluster with pgBackRest backup, provisioned entirely by OpenTofu (using `remote-exec` provisioners for OS-level bootstrap), and integrated into the existing secrets and service-discovery substrate.

---

## Constitutional Compliance

| Constitution | Constraint | Status |
|---|---|---|
| Org Art.3 | Architecture documentation required in techplan | ✅ This document |
| Terminus Art.1 | Features live in service repository boundary | ✅ Target repo: `terminus.infra` |
| Terminus Art.2 | Feature does not imply new repository | ✅ No new repo created |
| Terminus-Infra Art.1 | Infra features default to shared infra repo | ✅ `terminus.infra` is the home |
| Terminus-Infra Art.3 | Shared substrate owned centrally, not duplicated | ✅ PostgreSQL owned by infra, consumed by platform |

---

## Technology Stack

| Component | Technology | Version | Role |
|---|---|---|---|
| Database engine | PostgreSQL | 17.x (pin in techplan) | Primary RDBMS |
| HA manager | Patroni | Latest stable | Cluster lifecycle, automatic failover |
| DCS (consensus) | Consul | 1.22.5 (existing) | Patroni distributed state |
| VM provisioning | OpenTofu | 1.11.0 (existing) | Proxmox VM lifecycle |
| Cluster bootstrap | OpenTofu `remote-exec` | 1.11.0 (existing) | PostgreSQL + Patroni install + configure via SSH shell scripts |
| Backup | pgBackRest | Latest stable | WAL archiving, full/diff backup, restore |
| Credential storage | Vault KV v2 | 1.21.4 (existing) | All database credentials |
| Service discovery | Consul DNS | 1.22.5 (existing) | Primary endpoint resolution |
| IaC DB/role provider | OpenTofu `cyrilgdn/postgresql` | Latest stable | Database + role + grant management |
| IaC Vault provider | OpenTofu `hashicorp/vault` | Latest stable | Credential write-back to Vault |
| State backend | Local + SOPS/age | age (existing) | OpenTofu state encrypted at rest before git commit |
| Secret authoring | SOPS + age | Existing | Secret file encryption |

---

## Cluster Architecture

### Topology

```
┌─────────────────────────────────────────────────────────────────┐
│  Proxmox Host (single physical server, 32 cores / 256 GB RAM)   │
│                                                                  │
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │  postgres-primary-vm    │  │  postgres-replica-vm        │  │
│  │  4 vCPU / 16 GB RAM     │  │  4 vCPU / 16 GB RAM         │  │
│  │  32 GB OS + 150 GB data │  │  32 GB OS + 150 GB data     │  │
│  │                         │  │                             │  │
│  │  PostgreSQL 17.x        │  │  PostgreSQL 17.x            │  │
│  │  Patroni (primary role) │  │  Patroni (replica role)     │  │
│  │  pgBackRest             │  │  pgBackRest                 │  │
│  └────────────┬────────────┘  └─────────────┬───────────────┘  │
│               │  streaming replication        │                  │
│               └──────────────────────────────┘                  │
│                                                                  │
│  ┌──────────────────────────────────────┐                       │
│  │  postgres-backup-vm                  │                       │
│  │  2 vCPU / 4 GB RAM / 300 GB storage │                       │
│  │  pgBackRest repository host          │                       │
│  └──────────────────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────┘

  DCS / Service Discovery:
  ┌──────────────────────────────────────────┐
  │  Consul (existing cluster)               │
  │  - Patroni DCS leader election           │
  │  - Service: master.postgresql.service.consul → primary VM │
  └──────────────────────────────────────────┘

  OpenTofu State:
  ┌──────────────────────────────────────────┐
  │  Local state + SOPS/age encryption       │
  │  tofu/environments/{env}/terraform.tfstate.enc │
  │  Encrypted with operator age key before git commit │
  └──────────────────────────────────────────┘

  Secrets:
  ┌──────────────────────────────────────────┐
  │  Vault KV v2 (existing)                  │
  │  secret/terminus/{env}/postgres/...      │
  └──────────────────────────────────────────┘
```

### HA Failover Flow

1. Patroni on primary node fails health check
2. Patroni on replica acquires leader lock from Consul DCS
3. Replica promotes to primary (`pg_promote()`)
4. Consul service health check for `master.postgresql.service.consul` flips to new primary
5. Clients reconnect automatically — no operator intervention (FR12)
6. Operator reprovisioned lost VM via `tofu apply`; Patroni self-bootstraps as replica via Consul

**Failover detection window:** ~10-30 seconds (Patroni TTL-based, tunable)
**NFR1 compliance note:** The promotion window is the only unavailability window — no data loss (synchronous streaming replication on primary+replica).

---

## Service Discovery & Connection Model

### Client Connection Endpoints

| Endpoint | DNS | Purpose |
|---|---|---|
| Primary (read-write) | `master.postgresql.service.consul` | All writes; application default |
| Replica (read-only) | `replica.postgresql.service.consul` | Future read scaling; not used in MVP |

Patroni registers these service tags automatically via Consul integration. No manual DNS management. After failover, `master.postgresql.service.consul` resolves to the newly promoted primary within one Consul TTL cycle.

### Connection String Template

```
postgresql://{username}:{password}@master.postgresql.service.consul:5432/{dbname}?sslmode=require
```

All credentials sourced from Vault KV v2 — never hardcoded.

### Management Access (FR7)

FR7 is satisfied by the operator's existing **pgAdmin 4** installation on the workstation. No server-side management component is required or installed.

- pgAdmin 4 connects via TLS to `master.postgresql.service.consul:5432` (or direct VM IP during bootstrap before Consul is reachable)
- pgAdmin stores connection profiles locally on the operator workstation
- The Consul DNS endpoint provides post-failover resilience — pgAdmin reconnects to the new primary automatically
- No pgAdmin process runs on any cluster VM; firewall allows inbound 5432 from operator workstation IP

This is the definition of "management UI" used throughout the PRD and journey descriptions. No additional epic or server-side component is required.

### Future pgBouncer insertion

Insert pgBouncer as a transparent proxy in front of `master.postgresql.service.consul` without any client-side connection string changes. The DNS abstraction layer ensures NFR13 compliance — pgBouncer is not designed out.

---

## Provisioning Architecture

### Responsibility Split

| Layer | Tool | Scope |
|---|---|---|
| VM lifecycle | OpenTofu `bpg/proxmox` provider | Create, resize, destroy Proxmox VMs |
| PostgreSQL + Patroni bootstrap | OpenTofu `remote-exec` provisioner | Install packages, write `patroni.yml`, first-start via SSH shell scripts |
| pgBackRest bootstrap | OpenTofu `remote-exec` provisioner | Install pgBackRest, write `pgbackrest.conf`, `stanza-create` — runs in `postgres-backup` module after VM IP is known |
| Database / role / grants | OpenTofu `cyrilgdn/postgresql` provider | Idempotent DB objects via `tofu apply` |
| Credential delivery | OpenTofu `hashicorp/vault` provider | Write credentials to Vault KV v2 |

### OpenTofu Module Structure

Extends the existing `tofu/environments/{dev,prod}/` pattern in `terminus.infra`:

```
tofu/
  environments/
    dev/
        backend.tf          # local state — SOPS-encrypted (terraform.tfstate.enc) before git commit
        main.tf             # calls modules/postgres-cluster, modules/postgres-databases
        variables.tf
        versions.tf
    prod/
        backend.tf          # local state — SOPS-encrypted (terraform.tfstate.enc) before git commit
      main.tf
      variables.tf
      versions.tf
  modules/
    postgres-cluster/     # VM provisioning (Proxmox VMs + Patroni config output)
      main.tf
      variables.tf
      outputs.tf
    postgres-databases/   # Database objects (postgresql + vault providers)
      main.tf             # postgresql_database, postgresql_role, postgresql_grant, vault_kv_secret_v2
      variables.tf
      outputs.tf
    postgres-backup/      # Backup VM provisioning
      main.tf
      variables.tf
      outputs.tf
```

### Database Provisioning Pattern

Every database declaration follows this exact pattern — no variations:

```hcl
# 1. Database
resource "postgresql_database" "{dbname}" {
  name = "{dbname}"
}

# 2. Application role (minimum privilege)
resource "postgresql_role" "{dbname}_app" {
  name     = "{dbname}_app"
  login    = true
  password = random_password.{dbname}_app.result
}

# 3. Grants — CONNECT + CREATE on own database only
resource "postgresql_grant" "{dbname}_app_connect" {
  database    = postgresql_database.{dbname}.name
  role        = postgresql_role.{dbname}_app.name
  object_type = "database"
  privileges  = ["CONNECT", "CREATE"]
}

# 4. Credential delivery to Vault (FR4 schema)
resource "vault_kv_secret_v2" "{dbname}_creds" {
  mount = "secret"
  name  = "terminus/${var.environment}/postgres/{dbname}"
  data_json = jsonencode({
    username = postgresql_role.{dbname}_app.name
    password = random_password.{dbname}_app.result
    host     = "master.postgresql.service.consul"
    port     = 5432
    db_name  = postgresql_database.{dbname}.name
    ssl_mode = "require"
  })
}
```

**Idempotency:** `tofu apply` is a no-op if nothing changed. Re-running provisioning for an existing database does not regenerate credentials. EC9 resolved.

**Credential collision prevention:** `var.environment` is a required variable with bounded values (`dev`, `prod`). Vault path includes environment segment — no cross-environment collision possible.

---

## Backup Architecture

### pgBackRest Provisioning Order (Story Dependency)

**All pgBackRest bootstrap steps are OpenTofu `remote-exec` provisioners.** The ordering dependency is enforced via `depends_on` within the `postgres-backup` module:

1. `tofu apply` — provisions `postgres-backup` module → `postgres-backup-vm` exists, IP is known
2. `remote-exec` on cluster nodes — installs pgBackRest, writes `/etc/pgbackrest/pgbackrest.conf` (references backup VM IP output from step 1); depends on backup VM resource
3. `remote-exec` on backup VM — installs pgBackRest, writes `pgbackrest.conf`, runs `pgbackrest --stanza=main-{env} stanza-create`; depends on cluster node config (step 2)
4. `remote-exec` on backup VM — activates cron jobs for scheduled backups; depends on stanza existing (step 3)

All steps are OpenTofu. The `postgres-backup` module uses `depends_on` to enforce the cluster-node config → backup VM init sequence. The `postgres-cluster` and `postgres-backup` modules provision VMs in parallel; bootstrap `remote-exec` only runs after its own VM is up.

---

### pgBackRest Configuration

| Parameter | Value | Rationale |
|---|---|---|
| `archive-mode` | `on` | Enable WAL archiving |
| `archive-command` | `pgbackrest --stanza=main-${env} archive-push %p` | Push WAL to backup VM |
| `archive-async` | `y` | **NFR14 compliance** — async push never blocks primary |
| `archive-push-queue-max` | `4GB` | WAL queue on primary before error; sized for backup VM outage tolerance |
| `max_wal_size` | `2GB` | WAL accumulation limit; sized below queue-max |
| Backup schedule (full) | Weekly — Sunday 02:00 | Cron on backup VM |
| Backup schedule (diff) | Daily — 02:00 Mon-Sat | Cron on backup VM |
| Retention (full) | 2 full backups | ~2 weeks of full backup history |
| Retention (diff) | 14 diffs | 2-week diff history |
| Repository | `postgres-backup-vm:/var/lib/pgbackrest` | NFR9 — separate VM, not co-resident |
| Transport | SSH (key in Vault) | `secret/terminus/{env}/postgres/pgbackrest-ssh-key` |

### Backup Failure Handling (NFR14)

- `archive-async = y` means `archive_command` always returns immediately; WAL is spooled locally and pushed asynchronously
- If backup VM is unreachable: WAL spools on primary up to `archive-push-queue-max` (4GB); primary does not block
- `max_wal_size = 2GB` < `archive-push-queue-max = 4GB` — WAL recycling will not outpace queue capacity during a short backup VM outage
- Failed archive-push attempts produce log entries at `WARN` level — detectable without a monitoring solution (FR14 NFR14)

### Restore Procedure Overview

```bash
# Stop PostgreSQL on target
# Restore from latest backup
pgbackrest --stanza=main-${env} restore

# For PITR (future / growth phase):
pgbackrest --stanza=main-${env} restore \
  --target="2026-03-25 03:00:00" \
  --target-action=promote
```

---

## State & Vault Path Conventions

### OpenTofu State — SOPS-Local (ADR-0001 Deviation)

This capability deviates from ADR-0001 Consul state convention. State is stored locally and SOPS-encrypted before git commit, aligning with the PRD Architecture Constraints decision.

```
tofu/environments/dev/terraform.tfstate.enc   # SOPS-encrypted with operator age key
tofu/environments/prod/terraform.tfstate.enc  # SOPS-encrypted with operator age key
```

**Decrypt before apply:** `sops -d terraform.tfstate.enc > terraform.tfstate`
**Encrypt after apply:** `sops -e terraform.tfstate > terraform.tfstate.enc && rm terraform.tfstate`

> ADR-0001 Consul state convention applies to all other `terminus.infra` capabilities. Postgres uses SOPS-local per PRD Architecture Constraints. This deviation is intentional and documented here.

### Vault KV v2 Paths

| Path | Contents | Consumer |
|---|---|---|
| `secret/terminus/{env}/postgres/admin` | `{username, password}` superuser break-glass credential | Operator only |
| `secret/terminus/{env}/postgres/{dbname}` | `{username, password, host, port, db_name, ssl_mode}` | Consuming service |
| `secret/terminus/{env}/postgres/patroni-consul-token` | Consul ACL token for Patroni (if ACLs enabled) | Patroni process |
| `secret/terminus/{env}/postgres/pgbackrest-ssh-key` | SSH private key for backup VM access | pgBackRest on cluster nodes |
| `secret/terminus/{env}/postgres/replication` | `{username, password}` streaming replication role | Patroni replica bootstrap |

**Schema enforcement:** The `{dbname}` path schema `{username, password, host, port, db_name, ssl_mode}` is the consumer contract — all six fields always populated by the OpenTofu provisioning pattern. No field may be omitted.

---

## Ansible Playbook Structure

```
ansible/
  playbooks/
    bootstrap-postgres.yml      # Day-0: install PostgreSQL, Patroni, pgBackRest; init cluster
    configure-patroni.yml       # Patroni config templating (patroni.yml per node)
    configure-pgbackrest.yml    # pgBackRest config + stanza-create on backup VM
    validate-postgres.yml       # Acceptance test runner (FR18-20)
    maintain-postgres.yml       # Day-2: version upgrades, config changes
  roles/
    postgres/                   # PostgreSQL installation + base config
    patroni/                    # Patroni installation + config template
    pgbackrest/                 # pgBackRest installation + config
    postgres-hardening/         # TLS config, pg_hba.conf, audit logging
```

### Patroni Config Template (per-node variables)

```yaml
# /etc/patroni/patroni.yml (Ansible-templated)
scope: postgres-{{ environment }}
name: {{ inventory_hostname }}

consul:
  host: 127.0.0.1:8500
  token: "{{ vault_patroni_consul_token }}"  # from Vault

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB

postgresql:
  listen: 0.0.0.0:5432
  connect_address: "{{ ansible_host }}:5432"
  data_dir: /var/lib/postgresql/17/main
  parameters:
    wal_level: replica
    hot_standby: "on"
    max_wal_senders: 10
    max_replication_slots: 10
    archive_mode: "on"
    archive_command: >
      pgbackrest --stanza=main-{{ environment }} archive-push %p
    archive_cleanup_command: >
      pgbackrest --stanza=main-{{ environment }} archive-cleanup %r
    ssl: "on"
    ssl_cert_file: /etc/postgresql/tls/server.crt
    ssl_key_file: /etc/postgresql/tls/server.key
    ssl_ca_file: /etc/postgresql/tls/ca.crt
    log_destination: jsonlog
    logging_collector: "on"
    shared_preload_libraries: pgaudit
```

---

## TLS Configuration

All PostgreSQL connections require TLS (FR8, NFR4):

- Certificate authority: Terminus infra TLS CA (established in secrets feature, aligns with secrets D6)
- Server certificate: issued per-VM by infra TLS CA; stored on VM at `/etc/postgresql/tls/`
- Certificate provisioned during `bootstrap-postgres.yml` playbook; Vault PKI secrets engine or SOPS-managed cert bundle
- `ssl = on`, `ssl_ca_file` set — client must present valid TLS; plaintext connections rejected at listener via `pg_hba.conf` `hostssl` entries only

---

## Audit Logging (FR20, NFR12)

PostgreSQL `pgaudit` extension configured via `shared_preload_libraries`:

```sql
-- pgaudit settings (in postgresql.conf via Patroni DCS or Ansible)
pgaudit.log = 'ddl, write, role'   -- DDL, data writes, privilege changes
pgaudit.log_catalog = off           -- reduce noise from catalog queries
pgaudit.log_relation = on           -- log relation (table) access
pgaudit.log_statement_once = off
```

Log format: `log_destination = jsonlog` — structured JSON, compatible with future log aggregation pipeline (Loki, ELK, Splunk). Log rotation: `logrotate` configured for daily rotation, 30-day retention. Log location: `/var/log/postgresql/`.

---

## Acceptance Test Architecture (FR18-20)

Acceptance tests are Ansible-driven (part of `validate-postgres.yml`):

### Test 1 — Cluster Health (FR11)

```bash
patronictl -c /etc/patroni/patroni.yml list
# Assert: primary node state = "Leader", replica state = "Replica", lag = 0
```

### Test 2 — HA Failover (FR12)

```bash
patronictl -c /etc/patroni/patroni.yml switchover --force
# Assert: previous replica is now Leader within 60 seconds
# Assert: master.postgresql.service.consul resolves to new primary IP
patronictl -c /etc/patroni/patroni.yml switchover --force  # switch back
```

### Test 3 — DB Lifecycle + Isolation (FR18, FR19)

```python
# Automated test (Python psycopg2 + Vault SDK):
# 1. tofu apply with test database "acceptance_test_db" definition
# 2. Fetch credentials from Vault: secret/terminus/dev/postgres/acceptance_test_db
# 3. Connect as acceptance_test_app role; INSERT test row; SELECT row; assert row matches
# 4. Attempt connection to 'postgres' database as acceptance_test_app — assert FATAL: permission denied
# 5. tofu destroy target=module.postgres-databases.postgresql_database.acceptance_test_db
#    (terminates active connections via pg_terminate_backend first — EC6 resolution)
# Assert: database and role no longer exist; Vault secret deleted
```

### Test 4 — Credential Revocation (FR9)

```python
# 1. Capture current role password from Vault before rotation
before = vault.kv_get("secret/terminus/dev/postgres/acceptance_test_db")["password"]

# 2. Destroy and recreate the role via tofu apply (taint role resource)
#    pg_terminate_backend() called first to close active connections
#    tofu taint module.postgres-databases.postgresql_role.acceptance_test_db_app
#    tofu apply
new_creds = vault.kv_get("secret/terminus/dev/postgres/acceptance_test_db")

# Assert: password changed
assert new_creds["password"] != before

# Assert: old password rejected
try:
    psycopg2.connect(host="master.postgresql.service.consul", dbname="acceptance_test_db",
                     user="acceptance_test_db_app", password=before, sslmode="require")
    assert False, "Expected connection failure with old password"
except psycopg2.OperationalError:
    pass  # expected

# Assert: new password accepted
conn = psycopg2.connect(host="master.postgresql.service.consul", dbname="acceptance_test_db",
                        user=new_creds["username"], password=new_creds["password"], sslmode="require")
assert conn  # connected successfully
```

### Test 5 — Audit Log (FR20)

```bash
# After test 3, assert audit log contains:
grep '"object_type":"TABLE"' /var/log/postgresql/postgresql-*.json | grep acceptance_test_db
# Assert: connection event, INSERT event, FATAL denied event all present
```

### Test 5 — Backup Verification (FR13, FR17)

```bash
pgbackrest --stanza=main-dev backup --type=full
# Assert: exit code 0
pgbackrest --stanza=main-dev info
# Assert: backup set exists, size > 0, timestamp recent

# Restore test to isolated path
pgbackrest --stanza=main-dev restore --pg1-path=/tmp/pg-restore-test --target-action=promote
# Assert: PostgreSQL starts on restored data; known seeded row is readable (DoD data integrity check)
```

---

## Repository Structure (terminus.infra additions)

All assets live in `terminus.infra` per infra-constitution Art.1 and Art.2:

```
terminus.infra/
  tofu/
    environments/
      dev/
        main.tf               # calls postgres-cluster, postgres-databases, postgres-backup modules
        backend.tf            # Consul state: terminus/infra/postgres/dev/opentofu.tfstate
        versions.tf           # pins all provider versions
      prod/
        (mirrors dev)
    modules/
      postgres-cluster/       # VM provisioning (2 cluster VMs)
      postgres-databases/     # DB objects + Vault credential delivery
      postgres-backup/        # Backup VM provisioning
  ansible/
    playbooks/
      bootstrap-postgres.yml
      configure-patroni.yml
      configure-pgbackrest.yml
      validate-postgres.yml
      maintain-postgres.yml
    roles/
      postgres/
      patroni/
      pgbackrest/
      postgres-hardening/
  docs/
    decisions/
      adr-0001-proxmox-state-and-vault-conventions.md  # existing
      adr-0002-postgres-ha-tooling.md                  # NEW — Patroni + Consul
      adr-0003-postgres-backup-tooling.md              # NEW — pgBackRest
      adr-0004-postgres-provisioning-model.md          # NEW — OpenTofu providers
    runbooks/
      postgres-day0-bootstrap.md
      postgres-blast-and-repave.md
      postgres-backup.md
      postgres-restore.md
      postgres-new-database.md
      postgres-credential-rotation.md
      postgres-upgrade.md
      postgres-break-glass.md
  secrets/
    dev/
      postgres/               # SOPS-encrypted seed secrets (if any)
    prod/
      postgres/
  tests/
    postgres/
      acceptance/
        test_cluster_health.py
        test_db_lifecycle.py
        test_backup_verify.py
```

---

## Naming Conventions

All agents implementing this initiative must follow these patterns without variation:

| Concern | Convention | Example |
|---|---|---|
| VM hostname | `postgres-{role}-vm` | `postgres-primary-vm`, `postgres-replica-vm`, `postgres-backup-vm` |
| Patroni scope | `postgres-{env}` | `postgres-dev`, `postgres-prod` |
| pgBackRest stanza | `main-{env}` | `main-dev`, `main-prod` |
| Consul service tag (primary) | `master` | `master.postgresql.service.consul` |
| Consul service tag (replica) | `replica` | `replica.postgresql.service.consul` |
| OpenTofu module | `postgres-{concern}` | `postgres-cluster`, `postgres-databases`, `postgres-backup` |
| Vault credential path | `secret/terminus/{env}/postgres/{dbname}` | `secret/terminus/dev/postgres/temporal` |
| DB role name | `{dbname}_app` | `temporal_app` |
| Consul state key | `terminus/infra/postgres/{env}/opentofu.tfstate` | exact |
| ADR files | `adr-{nnnn}-{slug}.md` | `adr-0002-postgres-ha-tooling.md` |

---

## FR/NFR Coverage Validation

| Requirement | Architectural coverage | Status |
|---|---|---|
| FR1 | OpenTofu `tofu apply` provisions VMs + cluster; Ansible bootstraps | ✅ |
| FR2 | Blast-and-repave: `tofu apply` from git; Patroni self-bootstraps via Consul; preconditions documented | ✅ |
| FR3 | `modules/postgres-databases` OpenTofu module — idempotent, no VM access required | ✅ |
| FR4 | `vault_kv_secret_v2` resource writes full schema on every provisioning run | ✅ |
| FR5 | Vault KV v2 path schema documented; consumers use `vault kv get` or SDK | ✅ |
| FR6 | `postgresql_role` per database; `postgresql_grant` limits to CONNECT+CREATE on own DB | ✅ |
| FR7 | `patronictl list` + `psql` (no external UI required for MVP; management via CLI) | ✅ |
| FR8 | `pg_hba.conf` `hostssl` only; `ssl=on`; TLS cert from infra CA | ✅ |
| FR9 | `postgresql_role` + `vault_kv_secret_v2` destroyed and recreated; `pg_terminate_backend` pre-revocation | ✅ |
| FR10 | Patroni streaming replication (synchronous); primary failure → Patroni promotes replica | ✅ |
| FR7 | Remote pgAdmin 4 on operator workstation connects via TLS to `master.postgresql.service.consul:5432`; no server-side UI install required | ✅ |
| FR12 | Patroni automatic promotion via Consul leader election; no operator required | ✅ |
| FR13 | `pgbackrest backup --type=full` on demand | ✅ |
| FR14 | Cron: daily diff + weekly full on backup VM | ✅ |
| FR15 | `postgres-backup-vm` — separate VM, separate storage volumes from cluster VMs | ✅ |
| FR16 | `pgbackrest restore` + `tofu apply` + Patroni bootstrap; RTO ≤ 4h runbook | ✅ |
| FR17 | Test restore to `/tmp/pg-restore-test`; seeded row assertion | ✅ |
| FR18 | `validate-postgres.yml` playbook: provision, insert, verify, teardown | ✅ |
| FR19 | Acceptance test step 3: cross-database connection attempt asserts FATAL | ✅ |
| FR20 | `pgaudit` extension; JSON log; grep-based assertion in acceptance test | ✅ |
| FR21-23 | 8 runbooks in `docs/runbooks/`; all operations documented | ✅ |
| NFR1 | Patroni HA; 99.95% target — limited by VM-level failures only | ✅ |
| NFR2 | Daily diff backup; RPO ≤ 24h | ✅ |
| NFR3 | Runbook + `pgbackrest restore` + `tofu apply`; RTO ≤ 4h | ✅ |
| NFR4 | `ssl=on`; `pg_hba.conf` `hostssl` entries only | ✅ |
| NFR5 | `vault_kv_secret_v2` is only credential store; no plaintext in git | ✅ |
| NFR6 | `postgresql_role` per DB; superuser in Vault break-glass path | ✅ |
| NFR7 | `tofu apply` from git; Patroni self-bootstrap; bootstrap order documented | ✅ |
| NFR8 | `versions.tf` pins all providers; `tofu lock` file committed | ✅ |
| NFR9 | `postgres-backup-vm` separate VM; independent disk volumes | ✅ |
| NFR10 | 10 vCPU / 36 GB RAM total for postgres; <15% of 256 GB / 32 core host | ✅ |
| NFR11 | OpenTofu for VMs; OpenTofu for DB objects; Ansible for bootstrap config | ✅ |
| NFR12 | `pgaudit` + `jsonlog`; 30-day rotation via `logrotate` | ✅ |
| NFR13 | Patroni supports multi-standby; pgBackRest supports PITR with WAL archiving | ✅ |
| NFR14 | `archive-async=y`; `archive-push-queue-max=4GB`; `max_wal_size=2GB` | ✅ |

**All 23 FRs and 14 NFRs: ✅ architecturally covered.**

---

## Blast-and-Repave Sequence

Preconditions (from PRD NFR7):
1. Proxmox operational and reachable
2. Automation server (running OpenTofu) online and functional
3. Operator age key in custody
4. Network reachable between automation server and Proxmox
5. Vault operational (prerequisite — restore Vault before postgres if Vault also lost)

Sequence:
```
1. git checkout terminus.infra main
2. tofu -chdir=tofu/environments/{env} init
3. tofu -chdir=tofu/environments/{env} apply   # provisions VMs
4. ansible-playbook bootstrap-postgres.yml     # installs, configures, starts cluster
5. ansible-playbook configure-pgbackrest.yml   # stanza-create on backup VM
6. pgbackrest --stanza=main-{env} restore      # restore from latest backup (if data lost)
7. patronictl list                             # verify cluster healthy
8. ansible-playbook validate-postgres.yml      # run acceptance tests
```

---

## Open ADRs to Create During Implementation

Three decisions made in this architecture require formal ADRs in `terminus.infra/docs/decisions/`:

| ADR | Decision | Key rationale |
|---|---|---|
| ADR-0002 | Patroni + Consul as HA tooling | Consul already in stack; self-bootstrapping; no new DCS |
| ADR-0003 | pgBackRest as backup tooling | NFR14 async WAL; PITR-ready; Patroni-native integration |
| ADR-0004 | OpenTofu postgresql+vault providers for DB objects | Idempotency; single-tool provisioning chain; Vault schema contract |
