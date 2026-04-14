# Story 5.3: Runbook: Manual backup and restore

Status: ready-for-dev

## Story

As an operator, I want a runbook for on-demand backup, restore, and RTO verification, so that any operator can execute a recovery without prior context.

## Acceptance Criteria

1. Covers: manual backup trigger → backup verification → full restore procedure → data integrity check
2. Stored at `docs/terminus/infra/postgres/runbooks/03-backup-and-restore.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Run on-demand backup — `pgbackrest --stanza=main-dev backup --type=full`
  - [ ] Section 2: Verify backup — `pgbackrest --stanza=main-dev info` shows new backup entry
  - [ ] Section 3: Restore to isolated path — `pgbackrest --stanza=main-dev restore --pg1-path=/tmp/restore-test`
  - [ ] Section 4: Data integrity check — Start PostgreSQL instance from restored path; verify row count
  - [ ] Section 5: Full restore (cluster replacement) — Stop Patroni → clear PGDATA → `pgbackrest restore` → start Patroni
  - [ ] Section 6: Post-restore verification — `patronictl list` healthy, test query succeeds
- [ ] Write each section with copy-pasteable commands (AC: 1)
  - [ ] Include user context (run as `postgres` or `pgbackrest` OS user where applicable)
  - [ ] Include expected output for each verification step
- [ ] Commit runbook to `terminus.infra` (AC: 2)

## Dev Notes

### Architecture Context

Two restore scenarios are documented:
1. **Test restore to isolated path** (non-destructive) — used for integrity verification without affecting the running cluster
2. **Full cluster restore** — destructive; stops Patroni, clears PGDATA, restores from backup, restarts Patroni

The full cluster restore procedure is also the core of the blast-and-repave runbook (Story 5.2). Reference Story 5.2 for the full blast-and-repave variant.

- pgBackRest stanza: `main-dev`
- Run as: `postgres` OS user for cluster-level operations
- PGDATA: `/var/lib/postgresql/17/main`

### Responsibility Split

**This story does:**
- Authors the backup and restore runbook (both test restore and full restore paths)

**This story does NOT:**
- Switch over to a blast-and-repave (Story 5.2)
- Author the automated backup schedule (Stories 3.4)

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            03-backup-and-restore.md    # CREATE
```

### Testing Standards

- Runbook reviewed against actual pgBackRest behavior on the dev cluster
- All commands copy-pasteable; no ambiguous parameters
- Both restore paths (isolated test and full cluster restore) documented clearly

### References

- epics.md: Story 5.3 Acceptance Criteria
- architecture.md: FR13 — Manual backup on demand; FR16 — Restore within RTO ≤ 4h; FR17 — Test restore to isolated path
- architecture.md: NFR2 — RPO ≤ 24 hours; NFR3 — RTO ≤ 4 hours

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/03-backup-and-restore.md` — CREATE
