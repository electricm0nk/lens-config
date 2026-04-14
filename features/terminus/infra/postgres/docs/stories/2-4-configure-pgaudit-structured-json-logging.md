# Story 2.4: Configure pgaudit + structured JSON logging

Status: ready-for-dev

## Story

As an operator, I want pgaudit producing structured JSON logs capturing connections, DDL, and privilege escalations, so that NFR12 is satisfied and audit evidence is available.

## Acceptance Criteria

1. `pgaudit` extension loaded
2. `log_destination = jsonlog` in `postgresql.conf`
3. Connection event appears in JSON log after test connection
4. DDL event (CREATE TABLE) appears in JSON log
5. Log rotation configured via logrotate with 30-day retention

## Tasks / Subtasks

- [ ] Install `postgresql-17-pgaudit` package in bootstrap script (AC: 1)
  - [ ] Add to `bootstrap-postgres.sh` from Story 2.1 or separate pgaudit install step
- [ ] Configure `postgresql.conf` settings via `remote-exec` provisioner (AC: 1, 2)
  - [ ] `shared_preload_libraries = 'pgaudit'`
  - [ ] `log_destination = 'jsonlog'`
  - [ ] `logging_collector = on`
  - [ ] `log_directory = 'pg_log'`
  - [ ] `log_filename = 'postgresql-%Y-%m-%d.log'`
  - [ ] `pgaudit.log = 'all'` (or scoped to `ddl, role, connection`)
- [ ] Reload PostgreSQL after config changes (AC: 1, 2)
  - [ ] `patronictl reload postgres-{env}` after config write
- [ ] Configure `CREATE EXTENSION IF NOT EXISTS pgaudit` in Patroni bootstrap SQL (AC: 1)
- [ ] Write logrotate config for PostgreSQL JSON logs (AC: 5)
  - [ ] `/etc/logrotate.d/postgresql` — 30-day retention, compress, `postrotate: pg_ctlcluster reload`
- [ ] Verify log output after test events (AC: 3, 4)
  - [ ] Run test connection: check JSON log for connection event
  - [ ] Run `CREATE TABLE pgaudit_test (id int)`: check JSON log for DDL event
  - [ ] Run `DROP TABLE pgaudit_test`

## Dev Notes

### Architecture Context

`pgaudit` provides object-level and session-level audit logging for PostgreSQL. `log_destination = jsonlog` outputs structured JSON making logs compatible with future aggregation tools (Loki, Elasticsearch). Structured logging is a forward-compatibility requirement (NFR12).

- Log path: `$PGDATA/pg_log/postgresql-YYYY-MM-DD.log`
- `$PGDATA`: `/var/lib/postgresql/17/main`
- Log rotation: 30-day retention via logrotate
- `pgaudit.log` setting governs which statement classes are logged

### Responsibility Split

**This story does:**
- Installs and enables the `pgaudit` extension
- Configures `log_destination = jsonlog` and related logging settings
- Sets up logrotate for 30-day log retention
- Verifies audit events appear in JSON format

**This story does NOT:**
- Ship logs to a centralized aggregator (future initiative)
- Configure network-level logging or firewall logs
- Set up alerting on audit events

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf              # MODIFY — add pgaudit config provisioner
        scripts/
          configure-pgaudit.sh    # CREATE — postgresql.conf additions, logrotate config
          bootstrap-postgres.sh   # MODIFY — add postgresql-17-pgaudit package
```

### Testing Standards

- `psql -c "SELECT * FROM pg_extension WHERE extname = 'pgaudit'"` returns 1 row
- `SHOW shared_preload_libraries` includes `pgaudit`
- After test connection: `grep '"type":"connection"' $PGDATA/pg_log/postgresql-$(date +%Y-%m-%d).log` returns at least 1 line
- After `CREATE TABLE`: `grep '"command":"CREATE TABLE"' $PGDATA/pg_log/postgresql-$(date +%Y-%m-%d).log` returns at least 1 line
- `cat /etc/logrotate.d/postgresql` shows 30-day retention

### References

- epics.md: Story 2.4 Acceptance Criteria
- architecture.md: "Audit: pgaudit extension; log_destination=jsonlog; 30-day rotation via logrotate"
- architecture.md: NFR12 — Structured JSON audit logs capturing connections, DDL, privilege escalations
- architecture.md: FR20 — Operator can verify audit log shows all connection events, DDL operations, privilege escalation attempts

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (pgaudit config provisioner)
- `tofu/modules/postgres-cluster/scripts/configure-pgaudit.sh` — CREATE
- `tofu/modules/postgres-cluster/scripts/bootstrap-postgres.sh` — MODIFY (add pgaudit package)
