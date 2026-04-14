# Story 5.7: Runbook: Minor version upgrade (FR23)

Status: ready-for-dev

## Story

As an operator, I want a runbook for upgrading the PostgreSQL minor version with no manual VM changes, so that FR23 is satisfied and the procedure has been walked through at least once.

## Acceptance Criteria

1. Covers: update version pin in OpenTofu script → `tofu apply` (`remote-exec` re-runs package upgrade) → verify `SELECT version()` reports new version
2. Procedure validated at least once; outcome recorded in runbook
3. Stored at `docs/terminus/infra/postgres/runbooks/07-minor-version-upgrade.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Prerequisites — Identify new minor version; confirm new version available in apt repo
  - [ ] Section 2: Update version pin — Edit version string in `scripts/bootstrap-postgres.sh` or `versions.tf` comment
  - [ ] Section 3: Taint and apply — `tofu taint module.postgres_cluster.null_resource.bootstrap` (or equivalent); `tofu apply`
  - [ ] Section 4: `remote-exec` re-run — `apt-get install --only-upgrade postgresql-17=<new-version>` runs on both VMs
  - [ ] Section 5: Patroni restart — `patronictl restart postgres-dev` to pick up new binary
  - [ ] Section 6: Verify — `psql -c "SELECT version()"` on both nodes; confirm new minor version
- [ ] Write each section with copy-pasteable commands (AC: 1)
  - [ ] Note: minor version upgrade only — this procedure is not for major version upgrade (pg_upgrade required)
  - [ ] Include expected output after `SELECT version()`
- [ ] Execute procedure at least once (AC: 2)
  - [ ] May be executed with next available minor version; or validated against current version as a dry-run (no-op apply)
  - [ ] Record outcome in runbook Validation Record section
- [ ] Commit runbook to `terminus.infra` (AC: 3)

## Dev Notes

### Architecture Context

FR23 specifies minor version upgrade via OpenTofu `remote-exec` — the `apt-get install --only-upgrade` command runs on both VMs via the provisioner. A Patroni restart is required to pick up the new PostgreSQL binary. No `pg_upgrade` or data directory changes are needed for minor versions.

- Scope: PostgreSQL 17.x → 17.y (minor version only)
- Major version upgrade (17 → 18) requires a different procedure not covered here
- `patronictl restart postgres-dev` performs a rolling restart: replica first, then primary
- After upgrade, `pg_restart_pending` column in `patronictl list` should be false

### Responsibility Split

**This story does:**
- Authors the minor version upgrade runbook
- Validates the procedure at least once

**This story does NOT:**
- Upgrade Patroni or pgBackRest versions (those follow the same pattern but are separate)
- Cover major version upgrades

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            07-minor-version-upgrade.md    # CREATE
```

### Testing Standards

- Procedure executed at least once with recorded outcome
- `SELECT version()` output after upgrade shows expected minor version string
- Both VMs upgraded (verify on both primary and replica)
- Patroni shows no `pg_restart_pending` after restart

### References

- epics.md: Story 5.7 Acceptance Criteria
- architecture.md: FR23 — Operator can document and execute PostgreSQL minor version upgrade with no manual VM changes
- architecture.md: NFR8 — Pinned versions in OpenTofu lock files

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/07-minor-version-upgrade.md` — CREATE
