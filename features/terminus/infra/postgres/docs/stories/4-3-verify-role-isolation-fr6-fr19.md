# Story 4.3: Verify role isolation (FR6, FR19)

Status: ready-for-dev

## Story

As an operator, I want to confirm that `acceptance_test_db_app` cannot read or write to any database other than `acceptance_test_db`, so that FR6 and FR19 are verified.

## Acceptance Criteria

1. Connect as `acceptance_test_db_app`; attempt to connect to `postgres` database is denied
2. Attempt to CREATE TABLE in the `postgres` database is denied
3. SELECT on a table in another database is denied
4. All denials are confirmed in pgaudit JSON log

## Tasks / Subtasks

- [ ] Connect as `acceptance_test_db_app` to `postgres` database and confirm denial (AC: 1)
  - [ ] `psql "host=... dbname=postgres user=acceptance_test_db_app sslmode=require"` — expect `FATAL: permission denied for database postgres`
- [ ] Attempt CREATE TABLE in `postgres` database (AC: 2)
  - [ ] If connection to `postgres` is allowed (connect may succeed, create should fail), try: `CREATE TABLE isolation_test (id int)`
  - [ ] Confirm `ERROR: permission denied for schema public` or similar
- [ ] Attempt SELECT cross-database (AC: 3)
  - [ ] Connect as `acceptance_test_db_app` to `acceptance_test_db`
  - [ ] `SELECT * FROM postgres.public.some_table` — expect denial (cross-database queries not supported in PostgreSQL; document this as the enforcement mechanism)
- [ ] Verify denials appear in pgaudit JSON log (AC: 4)
  - [ ] Search JSON log for `acceptance_test_db_app` entries showing ERROR or FATAL events
  - [ ] Confirm `"error_severity":"ERROR"` or `"error_severity":"FATAL"` present for failed attempts
- [ ] Write acceptance test script (AC: 1, 2, 3, 4)
  - [ ] `tests/acceptance/test-role-isolation.sh` — scripted pass/fail test for all three attempt types

## Dev Notes

### Architecture Context

PostgreSQL's role isolation is enforced at two levels:
1. Database CONNECT privilege — `acceptance_test_db_app` has CONNECT only on `acceptance_test_db`
2. Cross-database access is architecturally impossible in PostgreSQL (no `db.schema.table` syntax) — document this as the mechanism

The test script should be reusable in acceptance Test 3 (Story 4.5).

- Role: `acceptance_test_db_app`
- Expected granted access: `acceptance_test_db` only
- pgaudit log path: `$PGDATA/pg_log/`

### Responsibility Split

**This story does:**
- Verifies role isolation for `acceptance_test_db_app`
- Creates a reusable isolation test script

**This story does NOT:**
- Provision the role (Story 4.1)
- Test credential rotation (Story 4.4)
- Run the full acceptance test suite (Story 4.5)

### Directory Layout

```
terminus.infra/
  tests/
    acceptance/
      test-role-isolation.sh    # CREATE — scripted role isolation verification
```

### Testing Standards

- Connection to `postgres` database as `acceptance_test_db_app` fails with permission error
- pgaudit JSON log contains ERROR/FATAL entries for `acceptance_test_db_app` failed attempts
- `test-role-isolation.sh` script exits 0 (PASS) when isolation is correctly enforced
- `test-role-isolation.sh` exits non-zero with clear error message if any isolation check fails

### References

- epics.md: Story 4.3 Acceptance Criteria
- architecture.md: FR6 — Each database has a dedicated service role scoped to that database only; no application workload uses superuser
- architecture.md: FR19 — Operator can verify that a service role cannot read or write outside its scoped database

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tests/acceptance/test-role-isolation.sh` — CREATE
