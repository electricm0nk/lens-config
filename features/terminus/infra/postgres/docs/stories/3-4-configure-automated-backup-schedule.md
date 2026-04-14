# Story 3.4: Configure automated backup schedule

Status: ready-for-dev

## Story

As an operator, I want full weekly and differential daily backups running automatically without operator intervention, so that FR14 is satisfied and RPO ≤ 24h (NFR2) is met.

## Acceptance Criteria

1. Cron job on backup VM runs full backup every Sunday 02:00 (`pgbackrest --stanza=main-{env} backup --type=full`)
2. Cron job on backup VM runs diff backup Mon–Sat 02:00 (`--type=diff`)
3. `pgbackrest --stanza=main-{env} info` shows at least one completed backup after manual trigger
4. 2-week retention policy configured (`repo1-retention-full=2`)

## Tasks / Subtasks

- [ ] Configure retention policy in `pgbackrest.conf` on backup VM (AC: 4)
  - [ ] `repo1-retention-full = 2` in `[global]` section
  - [ ] Update `scripts/bootstrap-pgbackrest-backup.sh` from Story 3.1
- [ ] Write cron jobs on backup VM via `remote-exec` provisioner (AC: 1, 2)
  - [ ] Full backup: `0 2 * * 0 pgbackrest --stanza=main-{env} backup --type=full` (Sunday 02:00)
  - [ ] Diff backup: `0 2 * * 1-6 pgbackrest --stanza=main-{env} backup --type=diff` (Mon-Sat 02:00)
  - [ ] Cron entries as `pgbackrest` or `postgres` OS user
  - [ ] Write to `/etc/cron.d/pgbackrest` for system-level cron
- [ ] Add `remote-exec` provisioner to `postgres-backup` module for cron setup (AC: 1, 2)
  - [ ] `scripts/configure-backup-schedule.sh` deployed via `remote-exec`
- [ ] Manually trigger a full backup and verify completion (AC: 3)
  - [ ] `sudo -u pgbackrest pgbackrest --stanza=main-{env} backup --type=full`
  - [ ] `pgbackrest --stanza=main-{env} info` shows backup with status `ok` and timestamp

## Dev Notes

### Architecture Context

Automated backups run from the backup VM, not from the cluster nodes. Cron is used rather than systemd timers for simplicity. The backup VM needs network access to the PostgreSQL primary via SSH (established in Story 3.1).

- Schedule: full weekly Sunday 02:00, diff daily Mon-Sat 02:00
- Retention: `repo1-retention-full = 2` (2 weeks of full backups, diffs automatically follow)
- Cron user: `pgbackrest` OS user (or `postgres`)
- Cron file: `/etc/cron.d/pgbackrest`

### Responsibility Split

**This story does:**
- Configures cron jobs for automated full + diff backup schedule
- Sets 2-week retention policy in `pgbackrest.conf`
- Verifies by triggering a manual backup

**This story does NOT:**
- Set up WAL archiving (Story 3.3)
- Test restore (Story 3.5)
- Set up monitoring/alerting for backup failures (future initiative)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-backup/
        main.tf              # MODIFY — add backup schedule provisioner
        scripts/
          configure-backup-schedule.sh   # CREATE — cron setup + retention config
          bootstrap-pgbackrest-backup.sh # MODIFY — add repo1-retention-full=2
```

### Testing Standards

- `ssh postgres-backup-vm 'cat /etc/cron.d/pgbackrest'` shows full and diff cron entries
- `ssh postgres-backup-vm 'sudo -u pgbackrest pgbackrest --stanza=main-dev backup --type=full'` exits 0
- `ssh postgres-backup-vm 'sudo -u pgbackrest pgbackrest --stanza=main-dev info'` shows 1+ completed backups
- `ssh postgres-backup-vm 'grep repo1-retention-full /etc/pgbackrest/pgbackrest.conf'` returns `repo1-retention-full=2`

### References

- epics.md: Story 3.4 Acceptance Criteria
- architecture.md: "full weekly (Sunday 02:00) + diff daily (Mon-Sat 02:00); 2-week retention"
- architecture.md: FR14 — Automated backups run daily without operator intervention
- architecture.md: NFR2 — RPO ≤ 24 hours (aligned to daily backup cadence)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-backup/main.tf` — MODIFY (backup schedule provisioner)
- `tofu/modules/postgres-backup/scripts/configure-backup-schedule.sh` — CREATE
- `tofu/modules/postgres-backup/scripts/bootstrap-pgbackrest-backup.sh` — MODIFY (add retention)
