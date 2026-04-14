# Story 3.1: Install and configure pgBackRest on all nodes

Status: ready-for-dev

## Story

As an operator, I want pgBackRest installed on all three VMs and `pgbackrest.conf` written on each via `remote-exec`, so that WAL archiving and backup jobs can route to the backup VM repository.

## Acceptance Criteria

1. pgBackRest installed on `postgres-primary-vm`, `postgres-replica-vm`, and `postgres-backup-vm`
2. `/etc/pgbackrest/pgbackrest.conf` written on each VM by `remote-exec` provisioner; backup VM IP sourced from `postgres-backup` module output
3. SSH key pair generated and stored in Vault (`secret/terminus/{env}/postgres/pgbackrest-ssh-key`); public key deployed to backup VM
4. `pgbackrest --stanza=main-{env} check` passes on primary node

## Tasks / Subtasks

- [ ] Generate pgBackRest SSH key pair and write to Vault (AC: 3)
  - [ ] Generate ed25519 key pair (or RSA 4096) for pgbackrest user
  - [ ] Write private key to `secret/terminus/{env}/postgres/pgbackrest-ssh-key` in Vault
  - [ ] Deploy public key to `~postgres/.ssh/authorized_keys` on backup VM via `remote-exec`
- [ ] Install pgBackRest package on all three VMs (AC: 1)
  - [ ] Add to `bootstrap-postgres.sh` or separate `bootstrap-pgbackrest.sh` script
  - [ ] Install `pgbackrest` at pinned version on primary, replica, and backup VMs
  - [ ] Create `pgbackrest` OS user if not created by package
- [ ] Write `/etc/pgbackrest/pgbackrest.conf` on cluster nodes (AC: 2)
  - [ ] `[global]` section: `repo1-host=<backup_vm_ip>`, `repo1-host-user=pgbackrest`
  - [ ] `[main-{env}]` stanza section: `pg1-path=/var/lib/postgresql/17/main`
  - [ ] Backup VM IP sourced from `module.postgres_backup.backup_vm_ip` output
- [ ] Write `/etc/pgbackrest/pgbackrest.conf` on backup VM (AC: 2)
  - [ ] `[global]` section: `repo1-path=/var/lib/pgbackrest`, `repo1-cipher-type=none`
  - [ ] `[main-{env}]` stanza section: `pg1-host=<primary_vm_ip>`, `pg1-path=/var/lib/postgresql/17/main`
- [ ] Run `pgbackrest --stanza=main-{env} check` on primary (AC: 4)
  - [ ] Execute as `postgres` user: `sudo -u postgres pgbackrest --stanza=main-{env} check`
  - [ ] Confirm exit code 0 (stanza not yet created — check validates config syntax, not stanza)

## Dev Notes

### Architecture Context

pgBackRest uses a dedicated backup VM (`postgres-backup-vm`) as its repository host. All cluster nodes communicate with the backup VM over SSH using a dedicated `pgbackrest` user. The backup VM IP must be known before `pgbackrest.conf` can be written on cluster nodes — this requires `module.postgres_backup` to be applied first (Story 1.2 dependency).

- pgBackRest stanza name: `main-{env}` — e.g., `main-dev`
- Backup VM path: `/var/lib/pgbackrest`
- SSH key Vault path: `secret/terminus/{env}/postgres/pgbackrest-ssh-key`
- Config path on all nodes: `/etc/pgbackrest/pgbackrest.conf`
- `archive-async = y` configured here (NFR14)

### Responsibility Split

**This story does:**
- Installs pgBackRest on all three VMs
- Writes `pgbackrest.conf` on all nodes with correct IP cross-references
- Sets up SSH key pair in Vault and deploys public key to backup VM
- Verifies config syntax via `pgbackrest check`

**This story does NOT:**
- Initialize the stanza (`stanza-create`) — Story 3.2
- Configure `archive_command` in PostgreSQL (Story 3.3)
- Set up backup schedules (Story 3.4)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf              # MODIFY — add pgbackrest install + conf provisioners on cluster nodes
        scripts/
          bootstrap-pgbackrest.sh     # CREATE — pgBackRest install + /etc/pgbackrest/pgbackrest.conf for cluster nodes
      postgres-backup/
        main.tf              # MODIFY — add pgbackrest install + conf + SSH authorized_keys provisioners
        scripts/
          bootstrap-pgbackrest-backup.sh   # CREATE — pgBackRest install + repo dir + pgbackrest.conf for backup VM
```

### Testing Standards

- `ssh postgres-primary-vm 'pgbackrest version'` returns pinned version
- `ssh postgres-backup-vm 'pgbackrest version'` returns pinned version
- `ssh postgres-primary-vm 'cat /etc/pgbackrest/pgbackrest.conf'` shows correct backup VM IP
- `ssh postgres-backup-vm 'cat /etc/pgbackrest/pgbackrest.conf'` shows correct primary VM IP
- `ssh postgres-primary-vm 'sudo -u postgres pgbackrest --stanza=main-dev check'` exits 0
- `vault kv get secret/terminus/dev/postgres/pgbackrest-ssh-key` returns a key value

### References

- epics.md: Story 3.1 Acceptance Criteria
- architecture.md: "Backup tooling: pgBackRest; archive-async=y; archive-push-queue-max=4GB"
- architecture.md: "Provisioning order dependency: tofu apply (all modules including postgres-backup) must complete before pgBackRest remote-exec provisioners run on cluster nodes"
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/pgbackrest-ssh-key"
- architecture.md: "pgBackRest stanza main-{env}"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (pgbackrest provisioners on cluster nodes)
- `tofu/modules/postgres-cluster/scripts/bootstrap-pgbackrest.sh` — CREATE
- `tofu/modules/postgres-backup/main.tf` — MODIFY (pgbackrest provisioners on backup VM)
- `tofu/modules/postgres-backup/scripts/bootstrap-pgbackrest-backup.sh` — CREATE
