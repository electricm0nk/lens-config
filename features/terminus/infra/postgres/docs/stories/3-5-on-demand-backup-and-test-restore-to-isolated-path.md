# Story 3.5: On-demand backup and test restore to isolated path

Status: ready-for-dev

## Story

As an operator, I want to run a manual backup on demand and restore it to an isolated path to verify integrity, so that FR13 and FR17 are satisfied and backup tooling is confirmed end-to-end before EPIC-004 acceptance tests.

## Acceptance Criteria

1. `pgbackrest --stanza=main-{env} backup --type=full` completes successfully on demand
2. Test restore to isolated path (`--pg1-path=/tmp/pgbackrest-restore-test`) succeeds
3. Restored data directory contains expected PostgreSQL cluster files
4. Original cluster is unaffected by the test restore

## Tasks / Subtasks

- [ ] Run on-demand full backup and confirm completion (AC: 1)
  - [ ] `sudo -u postgres pgbackrest --stanza=main-{env} backup --type=full`
  - [ ] `pgbackrest --stanza=main-{env} info` shows latest backup timestamp and status `ok`
- [ ] Execute test restore to isolated path (AC: 2, 3)
  - [ ] `sudo -u postgres pgbackrest --stanza=main-{env} restore --pg1-path=/tmp/pgbackrest-restore-test --delta`
  - [ ] Or use `--set` to restore specific backup label
  - [ ] Confirm restore exits 0
- [ ] Verify restored data directory (AC: 3)
  - [ ] `ls /tmp/pgbackrest-restore-test` shows: `PG_VERSION`, `global/`, `pg_wal/`, `base/`
  - [ ] `cat /tmp/pgbackrest-restore-test/PG_VERSION` shows `17`
- [ ] Verify original cluster unaffected (AC: 4)
  - [ ] `patronictl list` shows cluster still healthy after restore operation
  - [ ] `psql -h master.postgresql.service.consul -c "SELECT version()"` succeeds
- [ ] Clean up isolated restore path (AC: 4)
  - [ ] `rm -rf /tmp/pgbackrest-restore-test` after verification

## Dev Notes

### Architecture Context

This is a verification story. The test restore uses `--pg1-path=/tmp/pgbackrest-restore-test` to restore to a scratch directory without affecting the running cluster. The original PGDATA (`/var/lib/postgresql/17/main`) is untouched.

- This story does not start a PostgreSQL instance from the restored data directory
- The restore is a filesystem-level verification only (directory structure and `PG_VERSION` file)
- For a full data integrity verification, see EPIC-004 Story 4.6 (acceptance Test 6 — blast-and-repave + restore)

### Responsibility Split

**This story does:**
- Runs on-demand full backup
- Restores to isolated `/tmp` path
- Verifies directory structure
- Confirms original cluster unaffected

**This story does NOT:**
- Start a PostgreSQL instance from restored data (that is Story 4.6 / blast-and-repave)
- Test cluster recovery from backup in production-like scenario (EPIC-005)

### Directory Layout

```
terminus.infra/
  tests/
    acceptance/
      test-backup-restore.sh    # CREATE — scripted on-demand backup + isolated restore verification
```

### Testing Standards

- `pgbackrest backup --type=full` exits 0
- `pgbackrest restore --pg1-path=/tmp/pgbackrest-restore-test` exits 0
- `ls /tmp/pgbackrest-restore-test/PG_VERSION` exists and contains `17`
- `patronictl list` shows no change in cluster health after restore
- Script `tests/acceptance/test-backup-restore.sh` produces PASS output

### References

- epics.md: Story 3.5 Acceptance Criteria
- architecture.md: FR13 — Operator can perform manual backup on demand
- architecture.md: FR17 — Operator can verify backup integrity by executing a test restore to an isolated environment
- architecture.md: NFR3 — RTO ≤ 4 hours (full end-to-end test in acceptance Test 6)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tests/acceptance/test-backup-restore.sh` — CREATE
