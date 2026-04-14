# Story 2.1: Install PostgreSQL 17 + Patroni via OpenTofu remote-exec

Status: ready-for-dev

## Story

As an operator, I want PostgreSQL 17 and Patroni installed on both cluster VMs via OpenTofu `remote-exec` provisioners, so that the cluster nodes have all required packages before Patroni cluster initialization.

## Acceptance Criteria

1. PostgreSQL 17.x installed on both VMs, version pinned in `remote-exec` script
2. Patroni installed at pinned version on both VMs
3. `postgresql.service` disabled on both VMs (Patroni owns process lifecycle)
4. Prerequisite packages (python3-psycopg2, etc.) installed

## Tasks / Subtasks

- [ ] Write `remote-exec` bootstrap script for postgres packages (AC: 1, 3, 4)
  - [ ] Add PostgreSQL 17.x apt repository with pinned version string
  - [ ] Install: `postgresql-17`, `postgresql-client-17` at pinned version
  - [ ] Install prerequisites: `python3-psycopg2`, `python3-pip`, `libpq-dev`
  - [ ] Disable and stop `postgresql.service`: `systemctl disable postgresql && systemctl stop postgresql`
- [ ] Write `remote-exec` bootstrap script for Patroni (AC: 2)
  - [ ] Install Patroni at pinned version via pip or apt
  - [ ] Install `patroni[consul]` extras for Consul DCS support
  - [ ] Create `/etc/patroni/` directory and skeleton `patroni.yml`
- [ ] Add `remote-exec` provisioner blocks to `modules/postgres-cluster/main.tf` (AC: 1, 2, 3, 4)
  - [ ] `connection` block using SSH key from Vault or module variable
  - [ ] Inline provisioner or `file` provisioner for script delivery
- [ ] Pin all package versions in provisioner scripts (AC: 1, 2)
  - [ ] Document pinned versions in `versions.tf` comment or README
- [ ] Run `tofu apply` and verify on both VMs: `psql --version`, `patroni --version`, `systemctl is-enabled postgresql` (AC: 1, 2, 3, 4)

## Dev Notes

### Architecture Context

OpenTofu `remote-exec` replaces Ansible for all OS-level bootstrap in this initiative (architectural decision — Ansible removed from scope). Scripts run over SSH from the automation server during `tofu apply`.

- Module: `tofu/modules/postgres-cluster/`
- SSH connection: automation server → cluster VMs (key from Vault or module variable)
- Patroni scope: `postgres-{env}` (configured in Story 2.2)
- `postgresql.service` must be disabled — Patroni manages the PostgreSQL process

### Responsibility Split

**This story does:**
- Installs PostgreSQL 17.x and Patroni packages via `remote-exec`
- Disables the systemd `postgresql` service
- Installs all prerequisite packages

**This story does NOT:**
- Configure `patroni.yml` with Consul DCS settings (Story 2.2)
- Start the Patroni service or form a cluster (Story 2.2)
- Configure TLS (Story 2.3)
- Configure pgaudit (Story 2.4)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf        # MODIFY — add remote-exec provisioner blocks
        scripts/
          bootstrap-postgres.sh   # CREATE — PG17 + prerequisites install script
          bootstrap-patroni.sh    # CREATE — Patroni install script
```

### Testing Standards

- `tofu apply` completes with exit 0 on both VMs
- `ssh postgres-primary-vm 'psql --version'` returns `psql (PostgreSQL) 17.x`
- `ssh postgres-primary-vm 'patroni --version'` returns pinned Patroni version
- `ssh postgres-primary-vm 'systemctl is-enabled postgresql'` returns `disabled`
- `ssh postgres-replica-vm` same checks pass
- `tofu apply` re-run produces no changes (scripts idempotent)

### References

- epics.md: Story 2.1 Acceptance Criteria
- architecture.md: Technology Stack — PostgreSQL 17.x, Patroni latest stable
- architecture.md: "Cluster bootstrap: OpenTofu remote-exec — PostgreSQL + Patroni install + configure via SSH shell scripts"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (add remote-exec provisioner blocks)
- `tofu/modules/postgres-cluster/scripts/bootstrap-postgres.sh` — CREATE
- `tofu/modules/postgres-cluster/scripts/bootstrap-patroni.sh` — CREATE
