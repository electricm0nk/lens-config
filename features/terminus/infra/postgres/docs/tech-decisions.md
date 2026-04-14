---
initiative: terminus-infra-postgres
phase: techplan
date: "2026-03-25"
author: Todd
status: complete
---

# Technical Decisions Log — terminus-infra-postgres

This log records each architectural decision made during the TechPlan decision session, including the options considered, the choice made, and the rationale for future reference.

---

## Decision 1 — HA Tooling (OQ1)

**Question:** Which HA tooling handles Patroni cluster orchestration and leader election?

**Options considered:**

| Option | Pros | Cons |
|---|---|---|
| Patroni + Consul DCS | Consul already in stack; self-registering service discovery; native blast-and-repave story | Consul must remain operational |
| Patroni + etcd | Purpose-built DCS for Patroni | Introduces new substrate component (etcd) not in stack |
| Patroni + ZooKeeper | Battle-tested | Heavy; not in stack |
| repmgr | Simple | No automatic failover; requires manual promotion |

**Decision:** **Patroni + Consul DCS**

**Rationale:**
- Consul 1.22.5 is already deployed in the Terminus stack — no new infrastructure category
- Patroni's Consul integration handles service registration of `master.postgresql.service.consul` natively — no separate service registration step
- Consul provides leader lock TTL-based election; Patroni replica acquires lock on primary failure and promotes automatically
- Blast-and-repave is cleaner: after `tofu apply` (VMs + `remote-exec` bootstrap), Patroni finds the Consul key and self-describes as replica — no manual cluster re-join
- ADR-0002 to be created during implementation

**Status:** ACCEPTED

---

## Decision 2 — VM Sizing (OQ5)

**Question:** What VM resource allocation for postgres cluster nodes given actual host capacity?

**Host reality:** Single Proxmox server — 32 cores / 256 GB RAM

**Decision:**

| VM | vCPU | RAM | Disk |
|---|---|---|---|
| postgres-primary-vm | 4 | 16 GB | 32 GB OS + 150 GB data |
| postgres-replica-vm | 4 | 16 GB | 32 GB OS + 150 GB data |
| postgres-backup-vm | 2 | 4 GB | 300 GB repository storage |

**Rationale:**
- 4 vCPU/16 GB appropriate for a general-purpose homelab database serving multiple services
- 150 GB data disk per node: substantial headroom for Temporal workflows, application data growth
- Backup VM intentionally smaller — pgBackRest is not compute-intensive; 300 GB repository fits 2 weeks of backups for expected data volume
- Total postgres footprint: 10 vCPU / 36 GB RAM / ~482 GB storage — approximately 13% of available host RAM and 31% of host cores allocated to postgres tier
- NFR10 compliant: under 15% RAM threshold on 256 GB host

**Status:** ACCEPTED

---

## Decision 3 — Node Count (OQ6)

**Question:** How many postgres cluster nodes?

**Options considered:**

| Count | Notes |
|---|---|
| 3 nodes (primary + 2 replicas) | Standard enterprise recommendation; tolerates 1 node loss while maintaining quorum. Overkill for single-host HA. |
| 2 nodes (primary + 1 replica) | Standard two-node HA; sufficient for VM-level fault tolerance. Maximum effective replication on a single physical host. |
| 1 node (primary only) | No HA at all |

**Decision:** **2 nodes — primary + 1 replica**

**Rationale:**
- All VMs reside on the same physical host by design (Single Physical Host HA deviation, documented in PRD NFR9)
- Adding a third VM on the same host provides no additional resilience against the primary failure mode (host-level failure)
- Two-node topology fully satisfies NFR1 and the skills-transfer learning goal
- Primary + 1 synchronous replica is the textbook Patroni setup — reference materials, documentation, and community examples all target this topology

**Status:** ACCEPTED

---

## Decision 4 — Backup Tooling (OQ3)

**Question:** Which tool handles backup and restore? What backup schedule?

**Options considered:**

| Tool | Notes |
|---|---|
| pgBackRest | Native Patroni integration; async WAL archiving (NFR14); PITR-capable; repository host model fits separate backup VM |
| Barman | Feature-equivalent; similarly capable |
| pg_dump | Simple; no PITR; not suitable for HA cluster backup |
| Wal-E / Wal-G | Cloud-native WAL archiving; over-engineered for local backup VM |

**Decision:** **pgBackRest — async WAL archiving, full weekly + diff daily**

**Rationale:**
- `archive-async = y` mode satisfies NFR14 (archive_command must not block primary)
- Native Patroni integration: Patroni `archive_command` template uses pgBackRest stanza directly
- PITR-capable from day one: WAL archiving to backup VM enables point-in-time recovery (NFR13 future growth supported)
- Separate repository host (`postgres-backup-vm`) satisfies NFR9 — backup storage on independent VM with independent disk volumes
- Schedule: weekly full + daily diff provides RPO ≤ 24h (NFR2) with efficient storage use
- ADR-0003 to be created during implementation

**Status:** ACCEPTED

---

## Decision 5 — Database / Role Provisioning Model (OQ2)

**Question:** How are databases, roles, and grants provisioned? How are credentials delivered to Vault?

**Options considered:**

| Approach | Notes |
|---|---|
| OpenTofu `cyrilgdn/postgresql` + `hashicorp/vault` providers | Idempotent; single tool; Vault write integrated in same `tofu apply` |
| Ansible `community.postgresql` | Flexible; but not in-stack; separate tool chain from OpenTofu |
| SQL scripts in git | Simple; manual; not idempotent without extra logic |
| Custom shell scripts | Ad hoc; no state tracking |

**Decision:** **OpenTofu `cyrilgdn/postgresql` + `hashicorp/vault` providers**

**Rationale:**
- Single `tofu apply` execution provisions the database object, the role, the grants, and writes credentials to Vault — no multi-step manual process
- Idempotency: if role already exists with same password, no change occurs; EC9 resolved
- Vault write is atomic in the same Terraform state — credential path and role are always in sync
- FR3, FR4, FR5, FR6 all satisfied in one module definition
- ADR-0004 to be created during implementation

**Status:** ACCEPTED

---

## Decision 6 — Connection Model (OQ4)

**Question:** How do consuming services connect to the cluster? Do we use a connection pooler?

**Options considered:**

| Model | Notes |
|---|---|
| Direct TCP via Consul DNS (`master.postgresql.service.consul`) | Zero new components; Patroni registers natively; connection string is static after failover |
| pgBouncer in front of Consul endpoint | Adds connection pooling; more complexity for MVP |
| HAProxy | Load-balancer model; typical in cloud Postgres setups; not needed for homelab scale |

**Decision:** **Direct TCP via `master.postgresql.service.consul`**

**Rationale:**
- Patroni registers the primary node as a Consul service on port 5432 automatically; no operator DNS management
- After failover, `master.postgresql.service.consul` resolves to the new primary within one Consul TTL cycle
- Clients use a static connection string — no application reconfiguration on failover
- Connection string: `postgresql://{user}:{pass}@master.postgresql.service.consul:5432/{dbname}?sslmode=require`
- pgBouncer is architecturally insertable later (between client and `master.postgresql.service.consul`) without any consumer-side changes — NFR13 not violated

**Status:** ACCEPTED

---

## Decision 7 — State and Vault Path Conventions

**Question:** How should Consul state keys and Vault paths be structured for postgres, given ADR-0001?

**ADR-0001 base patterns:**
- State: `terminus/infra/{capability}/{env}/opentofu.tfstate` (Consul)
- Vault: `secret/terminus/{env}/{capability}/{secret-name}`

**Decision — Deviates from ADR-0001 for state only:**

Postgres uses SOPS-local state per PRD Architecture Constraints. Consul state convention (ADR-0001) applies to all other `terminus.infra` capabilities; postgres explicitly deviates and documents the reason here.

**Decision:**

| Path type | Pattern |
|---|---|
| OpenTofu state (dev) | `tofu/environments/dev/terraform.tfstate.enc` (SOPS-local) |
| OpenTofu state (prod) | `tofu/environments/prod/terraform.tfstate.enc` (SOPS-local) |
| Vault credential (per DB) | `secret/terminus/{env}/postgres/{dbname}` |
| Vault admin credential | `secret/terminus/{env}/postgres/admin` |
| Vault replication credential | `secret/terminus/{env}/postgres/replication` |
| Vault pgBackRest SSH key | `secret/terminus/{env}/postgres/pgbackrest-ssh-key` |
| Vault Patroni Consul token | `secret/terminus/{env}/postgres/patroni-consul-token` |

**Vault credential schema (per-DB path):**
```json
{
  "username": "{dbname}_app",
  "password": "<generated>",
  "host": "master.postgresql.service.consul",
  "port": 5432,
  "db_name": "{dbname}",
  "ssl_mode": "require"
}
```

**Rationale:**
- State: SOPS-local aligns with PRD Architecture Constraints; operator age key already in custody; solo-operator environment means concurrent apply conflicts are not a concern
- Vault: follows ADR-0001 Vault path conventions without deviation
- `{env}` segment prevents cross-environment credential collision
- Full six-field schema is the consumer contract — all fields always populated by provisioning

**Status:** ACCEPTED
