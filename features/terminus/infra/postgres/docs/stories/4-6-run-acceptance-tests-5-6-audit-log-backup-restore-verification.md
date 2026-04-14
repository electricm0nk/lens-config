# Story 4.6: Run acceptance Tests 5–6 (audit log, backup/restore verification)

Status: ready-for-dev

## Story

As an operator, I want the final two automated acceptance tests to pass, so that audit logging and backup/restore RTO ≤ 4h are formally verified.

## Acceptance Criteria

1. **Test 5 (Audit Log):** Connection event, DDL (CREATE TABLE), and privilege escalation attempt all appear in JSON audit log
2. **Test 6 (Backup/Restore Verification):** Full backup taken → cluster VMs destroyed → `tofu apply` reprovisioning → restore from backup → `acceptance_test_db` data intact; total elapsed time ≤ 4h (NFR3, FR16)

## Tasks / Subtasks

- [ ] Implement Test 5: Audit Log Verification (AC: 1)
  - [ ] Connect as `acceptance_test_db_app` — assert connection event in JSON log
  - [ ] `CREATE TABLE audit_test (id int)` — assert DDL event in JSON log
  - [ ] Attempt `SET ROLE postgres` or `ALTER SYSTEM` — assert privilege escalation denial in log
  - [ ] Script: `tests/acceptance/test-audit-log.sh`
- [ ] Implement Test 6: Backup/Restore End-to-End (AC: 2)
  - [ ] Pre-test: insert known data into `acceptance_test_db` — `INSERT INTO restore_verification VALUES ('test-data-$(date)')`
  - [ ] Run full backup: `pgbackrest --stanza=main-{env} backup --type=full`
  - [ ] Destroy cluster VMs: `tofu destroy -target module.postgres_cluster` (VMs only, not backup VM)
  - [ ] Reprovision: `tofu apply` — cluster VMs reprovisioned, Patroni bootstrapped
  - [ ] Restore from backup: `pgbackrest --stanza=main-{env} restore` (PGDATA path)
  - [ ] Start cluster and verify: `patronictl list` shows healthy cluster
  - [ ] Assert data: `psql -c "SELECT * FROM restore_verification"` returns pre-destroy rows
  - [ ] Record total elapsed time — assert ≤ 4 hours
- [ ] Add Test 5 and Test 6 to `run-acceptance-tests.sh` orchestrator (AC: 1, 2)
- [ ] Run all tests and record outcome (AC: 1, 2)

## Dev Notes

### Architecture Context

Test 6 is the most complex acceptance test — it validates the full blast-and-repave + restore cycle end-to-end. This directly verifies NFR3 (RTO ≤ 4h) and FR16 (restore from backup within RTO). The backup VM and its data must be preserved during the VM destroy step.

**CRITICAL:** `tofu destroy` must target ONLY the cluster VMs (`module.postgres_cluster`), NOT the backup VM. The backup VM holds the pgBackRest repository.

- Runbook for this procedure: Story 5.2 (blast-and-repave runbook)
- Elapsed time tracking: `date` before and after Test 6
- Data assertion: `acceptance_test_db.restore_verification` table with timestamped rows

### Responsibility Split

**This story does:**
- Implements and runs Tests 5 and 6
- Full cluster destroy + reprovision + restore cycle

**This story does NOT:**
- Implement the blast-and-repave runbook (Story 5.2 documents the procedure executed here)
- Modify backup infrastructure (EPIC-003)

### Directory Layout

```
terminus.infra/
  tests/
    acceptance/
      run-acceptance-tests.sh    # MODIFY — add Test 5 and Test 6 invocations
      test-audit-log.sh          # CREATE — Test 5
      test-backup-restore-e2e.sh # CREATE — Test 6 (blast-and-repave + restore)
```

### Testing Standards

- Test 5: JSON log grep for connection/DDL/privilege events — all three found, script exits 0
- Test 6: Total elapsed time from destroy to verified data recovery ≤ 4 hours, recorded in completion notes
- `run-acceptance-tests.sh 5 6` exits 0 with both tests PASS
- Post-Test 6: cluster is fully functional (patronictl shows Leader + Replica healthy)

### References

- epics.md: Story 4.6 Acceptance Criteria
- architecture.md: FR16 — Restore from backup + RTO ≤ 4h; FR20 — Audit log verification
- architecture.md: NFR3 — RTO ≤ 4 hours from declared incident to verified operational cluster
- architecture.md: NFR12 — Structured JSON audit logs

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tests/acceptance/run-acceptance-tests.sh` — MODIFY (add Test 5, Test 6)
- `tests/acceptance/test-audit-log.sh` — CREATE
- `tests/acceptance/test-backup-restore-e2e.sh` — CREATE
