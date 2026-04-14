# Story 4.5: Run acceptance Tests 1–3 (cluster health, DB lifecycle, role isolation)

Status: ready-for-dev

## Story

As an operator, I want the first three automated acceptance tests to pass, so that cluster health, database lifecycle, and role isolation are formally verified.

## Acceptance Criteria

1. **Test 1 (Cluster Health):** `patronictl list` shows Leader + Replica healthy; replication lag = 0
2. **Test 2 (DB Lifecycle):** Provision test DB → insert row → verify row readable → drop DB → verify gone; all via OpenTofu + psycopg2 script
3. **Test 3 (Role Isolation):** `acceptance_test_db_app` denied access to all other databases (automated script, mirrors Story 4.3)

## Tasks / Subtasks

- [ ] Write / finalize acceptance test runner script (AC: 1, 2, 3)
  - [ ] `tests/acceptance/run-acceptance-tests.sh` — orchestrates Test 1, 2, 3
  - [ ] Each test: outputs PASS/FAIL to stdout with test name
  - [ ] Non-zero exit on any FAIL
- [ ] Implement Test 1: Cluster Health (AC: 1)
  - [ ] Parse `patronictl -c /etc/patroni/patroni.yml list` output
  - [ ] Assert: 1 Leader + 1 Replica, both Running
  - [ ] Assert: replication lag = 0 (or within acceptable threshold, e.g., < 1MB)
- [ ] Implement Test 2: DB Lifecycle (AC: 2)
  - [ ] OpenTofu: create a transient `lifecycle_test_db` via `postgres-databases` module
  - [ ] psycopg2 Python script: connect as `lifecycle_test_db_app`, `CREATE TABLE t (id int)`, `INSERT INTO t VALUES (42)`, `SELECT id FROM t WHERE id = 42` — assert returns 42
  - [ ] OpenTofu: destroy `lifecycle_test_db` via `tofu destroy -target module.lifecycle_test_db`
  - [ ] Assert: database no longer listed in `\l`
- [ ] Implement Test 3: Role Isolation (AC: 3)
  - [ ] Reuse `tests/acceptance/test-role-isolation.sh` from Story 4.3
  - [ ] Wrap in test runner with PASS/FAIL output
- [ ] Run all three tests against dev cluster (AC: 1, 2, 3)
  - [ ] Record run output in story completion notes

## Dev Notes

### Architecture Context

Acceptance tests run from the automation server against the running dev cluster. Tests 1–3 cover the cluster EPIC-002 and database EPIC-004 stories in an automated, repeatable way.

- psycopg2 available on automation server (install if missing: `pip3 install psycopg2-binary`)
- OpenTofu on automation server: `tofu` command in PATH
- Vault CLI on automation server: `vault` command for credential retrieval
- Test 2 uses a transient database specifically for the lifecycle test — this avoids polluting `acceptance_test_db`

### Responsibility Split

**This story does:**
- Implements and runs Tests 1, 2, 3
- Creates `tests/acceptance/run-acceptance-tests.sh` orchestrator

**This story does NOT:**
- Run Tests 4 (credential rotation — Story 4.4) or Tests 5–6 (Story 4.6)
- Create permanent databases beyond the transient lifecycle_test_db

### Directory Layout

```
terminus.infra/
  tests/
    acceptance/
      run-acceptance-tests.sh      # CREATE — orchestrator for all 6 tests
      test-cluster-health.sh       # CREATE — Test 1
      test-db-lifecycle.py         # CREATE — Test 2 (psycopg2)
      test-role-isolation.sh       # EXISTING from Story 4.3 — Test 3
```

### Testing Standards

- `run-acceptance-tests.sh 1 2 3` exits 0 with all three tests reporting PASS
- Test 2 `lifecycle_test_db` database is absent from `\l` after test completes
- Test runner output is human-readable with clear PASS/FAIL indicators per test number

### References

- epics.md: Story 4.5 Acceptance Criteria
- architecture.md: FR10 — HA single node failure; FR11 — HA status verifiable; FR18 — Automated acceptance test
- architecture.md: FR19 — Role isolation

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tests/acceptance/run-acceptance-tests.sh` — CREATE
- `tests/acceptance/test-cluster-health.sh` — CREATE
- `tests/acceptance/test-db-lifecycle.py` — CREATE
- `tests/acceptance/test-role-isolation.sh` — EXISTING (from Story 4.3)
