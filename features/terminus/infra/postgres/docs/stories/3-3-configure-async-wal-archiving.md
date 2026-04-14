# Story 3.3: Configure async WAL archiving

Status: ready-for-dev

## Story

As an operator, I want WAL archiving enabled with `archive-async=y`, so that the primary is never blocked by backup VM outage (NFR14).

## Acceptance Criteria

1. `archive_mode = on` and `archive_command` set in `postgresql.conf`
2. `archive-async = y` and `archive-push-queue-max = 4GB` in `pgbackrest.conf`
3. WAL segment appears in backup repository after test transaction
4. Simulated backup VM pause does not cause primary to stall (confirmed via log inspection)

## Tasks / Subtasks

- [ ] Configure `archive_mode` and `archive_command` in `postgresql.conf` (AC: 1)
  - [ ] `archive_mode = on`
  - [ ] `archive_command = 'pgbackrest --stanza=main-{env} archive-push %p'`
  - [ ] Delivered via Patroni config or `patronictl edit-config` / `remote-exec` provisioner
- [ ] Configure async settings in `pgbackrest.conf` on cluster nodes (AC: 2)
  - [ ] `archive-async = y`
  - [ ] `archive-push-queue-max = 4GB`
  - [ ] `max_wal_size = 2GB` in `postgresql.conf` (architecture spec)
  - [ ] Update `scripts/bootstrap-pgbackrest.sh` from Story 3.1 to include these settings
- [ ] Reload PostgreSQL to activate archive settings (AC: 1)
  - [ ] `patronictl reload postgres-{env}` or `SELECT pg_reload_conf()`
  - [ ] `archive_mode = on` requires a full restart if not already enabled — plan for `patronictl restart`
- [ ] Verify WAL archiving is active (AC: 3)
  - [ ] Insert test rows: `psql -c "INSERT INTO archive_test SELECT generate_series(1,1000)"`
  - [ ] Force WAL rotation: `psql -c "SELECT pg_switch_wal()"`
  - [ ] Confirm WAL segment in backup repository: `pgbackrest --stanza=main-{env} info` shows archived WAL
- [ ] Simulate backup VM unavailability and confirm primary does not stall (AC: 4)
  - [ ] Stop pgbackrest service or block network on backup VM temporarily
  - [ ] Execute writes on primary and confirm query latency is unaffected
  - [ ] Re-enable backup VM; confirm async queue drains to repository

## Dev Notes

### Architecture Context

Async WAL archiving (`archive-async=y`) is required by NFR14 — backup failures must not cause the primary to stall. pgBackRest's async push queues WAL locally up to `archive-push-queue-max` before failing. This decouples primary availability from backup VM availability.

- `archive_mode = on` requires a PostgreSQL restart (not just reload) to take effect
- `archive_command` must be configured before stanza is created — or set after; pgBackRest handles ordering
- `max_wal_size = 2GB` limits WAL generation rate
- Patroni manages `postgresql.conf` — use `patronictl edit-config` or ensure the setting is in the static config

### Responsibility Split

**This story does:**
- Configures `archive_mode`, `archive_command`, `archive-async`, and `archive-push-queue-max`
- Verifies WAL segments appear in repository
- Verifies async non-blocking behavior

**This story does NOT:**
- Set up backup schedules (Story 3.4)
- Run full backups (Story 3.5)
- Initialize the stanza (Story 3.2)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf              # MODIFY — add archive settings provisioner
        scripts/
          configure-wal-archiving.sh   # CREATE — postgresql.conf archive settings + reload
          bootstrap-pgbackrest.sh      # MODIFY — add archive-async, archive-push-queue-max
```

### Testing Standards

- `SHOW archive_mode` returns `on`
- `SHOW archive_command` returns pgbackrest push command string
- `SELECT pg_switch_wal()` followed by `pgbackrest --stanza=main-dev info` shows WAL archived
- Primary query latency unaffected during 60-second backup VM network block (log inspection confirms async queue used)

### References

- epics.md: Story 3.3 Acceptance Criteria
- architecture.md: "archive-async=y; archive-push-queue-max=4GB; max_wal_size=2GB"
- architecture.md: NFR14 — archive_command must be async; backup failures produce detectable log entries; primary must not stall on backup VM outage

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (archive settings provisioner)
- `tofu/modules/postgres-cluster/scripts/configure-wal-archiving.sh` — CREATE
- `tofu/modules/postgres-cluster/scripts/bootstrap-pgbackrest.sh` — MODIFY (async settings)
