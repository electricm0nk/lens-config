# Story 3.2: Initialize pgBackRest stanza

Status: ready-for-dev

## Story

As an operator, I want the pgBackRest stanza `main-{env}` created on the backup VM, so that the backup repository is initialized and ready to accept WAL archives and backups.

## Acceptance Criteria

1. `pgbackrest --stanza=main-{env} stanza-create` completes successfully on backup VM (run via `remote-exec` after cluster node config in Story 3.1)
2. Repository directory structure exists at `/var/lib/pgbackrest`
3. `pgbackrest --stanza=main-{env} info` returns stanza status without error

## Tasks / Subtasks

- [ ] Add `remote-exec` provisioner to backup VM module to run `stanza-create` (AC: 1)
  - [ ] `sudo -u pgbackrest pgbackrest --stanza=main-{env} stanza-create`
  - [ ] Provisioner must have `depends_on` = cluster node pgbackrest config provisioners (Story 3.1)
- [ ] Verify repository directory structure created (AC: 2)
  - [ ] `ls /var/lib/pgbackrest` shows expected pgBackRest repository subdirectories
- [ ] Run `pgbackrest info` and confirm stanza reports successfully (AC: 3)
  - [ ] `sudo -u pgbackrest pgbackrest --stanza=main-{env} info` exits 0
  - [ ] Output includes `stanza: main-{env}` and `status: ok` (or similar status)

## Dev Notes

### Architecture Context

`stanza-create` initializes the backup repository on the backup VM. It must run after:
1. pgBackRest is installed on all nodes (Story 3.1)
2. pgBackRest SSH key is deployed (Story 3.1)
3. pgBackRest is configured on cluster nodes (Story 3.1)
4. PostgreSQL cluster is running (Story 2.2)

The `stanza-create` command contacts the PostgreSQL primary to collect catalog information, so the PostgreSQL cluster must be operational before this step.

- Stanza name: `main-{env}` — e.g., `main-dev`
- Executed as: `pgbackrest` OS user (or `postgres`)
- Repository root: `/var/lib/pgbackrest`
- Ordering in `tofu/modules/postgres-backup/main.tf`: `null_resource` ordered after cluster provisioners complete

### Responsibility Split

**This story does:**
- Runs `stanza-create` via `remote-exec` on the backup VM
- Verifies repository structure and stanza info

**This story does NOT:**
- Run any backups (Story 3.4, 3.5)
- Configure WAL archiving in PostgreSQL (Story 3.3)
- Install pgBackRest (Story 3.1)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-backup/
        main.tf    # MODIFY — add stanza-create remote-exec provisioner with depends_on
```

### Testing Standards

- `ssh postgres-backup-vm 'sudo -u pgbackrest pgbackrest --stanza=main-dev stanza-create'` exits 0
- `ssh postgres-backup-vm 'ls /var/lib/pgbackrest'` shows repository structure (archive, backup subdirs)
- `ssh postgres-backup-vm 'sudo -u pgbackrest pgbackrest --stanza=main-dev info'` exits 0 and shows stanza status
- `tofu apply` re-run with stanza already created: no error (command is idempotent for existing stanzas)

### References

- epics.md: Story 3.2 Acceptance Criteria
- architecture.md: "stanza-create runs after backup VM IP is known (module output) and cluster node config is complete"
- architecture.md: pgBackRest stanza `main-{env}`

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-backup/main.tf` — MODIFY (add stanza-create remote-exec provisioner)
