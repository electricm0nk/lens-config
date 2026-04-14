# Story 4.2: Provision `acceptance_test_db` and verify Vault credentials

Status: ready-for-dev

## Story

As an operator, I want to provision the demo database `acceptance_test_db` and retrieve its credentials from Vault, so that FR3, FR4, and FR5 are exercised end-to-end.

## Acceptance Criteria

1. `acceptance_test_db` database exists in the cluster
2. `acceptance_test_db_app` role exists with login privilege and no superuser rights
3. Full credential bundle readable from `secret/terminus/{env}/postgres/acceptance_test_db`
4. `psql` connection using Vault credentials succeeds with `sslmode=require`

## Tasks / Subtasks

- [ ] Confirm `module "acceptance_test_db"` applied from Story 4.1 (AC: 1, 2)
  - [ ] `psql -h master.postgresql.service.consul -U postgres -c "\l"` shows `acceptance_test_db`
  - [ ] `psql -h master.postgresql.service.consul -U postgres -c "\du"` shows `acceptance_test_db_app`
- [ ] Verify role properties (AC: 2)
  - [ ] `SELECT rolsuper, rolcreatedb, rolcreaterole, rolcanlogin FROM pg_roles WHERE rolname = 'acceptance_test_db_app'`
  - [ ] Confirm: `rolsuper=f`, `rolcanlogin=t`
- [ ] Retrieve credential bundle from Vault (AC: 3)
  - [ ] `vault kv get secret/terminus/{env}/postgres/acceptance_test_db`
  - [ ] Confirm all 6 fields present: `username`, `password`, `host`, `port`, `db_name`, `ssl_mode`
  - [ ] `ssl_mode` value is `require`
- [ ] Connect using Vault credentials with TLS (AC: 4)
  - [ ] Extract `PGPASSWORD` from Vault output
  - [ ] `psql "host=master.postgresql.service.consul port=5432 dbname=acceptance_test_db user=acceptance_test_db_app sslmode=require"` succeeds
  - [ ] Confirm connected user: `SELECT current_user` returns `acceptance_test_db_app`

## Dev Notes

### Architecture Context

This story verifies the end-to-end flow of the `postgres-databases` module introduced in Story 4.1. All work is operational verification against the running cluster — no new OpenTofu resources.

- Vault path: `secret/terminus/dev/postgres/acceptance_test_db`
- Role name: `acceptance_test_db_app`
- Connection: `master.postgresql.service.consul:5432` with `sslmode=require`

### Responsibility Split

**This story does:**
- Verifies database creation, role creation, and Vault credential write-back
- Tests end-to-end credential consumption via `psql`

**This story does NOT:**
- Provision the database (Story 4.1)
- Test role isolation (Story 4.3)
- Test credential rotation (Story 4.4)

### Directory Layout

No new `terminus.infra` files. Verification is performed live against the cluster.

### Testing Standards

- `psql` connection with Vault credentials and `sslmode=require` exits 0
- `SELECT current_user` inside the connection returns `acceptance_test_db_app`
- `vault kv get` output shows all 6 credential fields
- `pg_roles` query confirms no superuser rights on `acceptance_test_db_app`

### References

- epics.md: Story 4.2 Acceptance Criteria
- architecture.md: FR3 — Provision database + role via automation
- architecture.md: FR4 — Credentials auto-written to Vault KV v2
- architecture.md: FR5 — Retrieve credentials from Vault KV v2
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/{dbname} — username, password, host, port, db_name, ssl_mode"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

(no terminus.infra files — verification only)
