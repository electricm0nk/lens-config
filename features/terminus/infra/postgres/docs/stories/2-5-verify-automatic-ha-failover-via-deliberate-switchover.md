# Story 2.5: Verify automatic HA failover via deliberate switchover

Status: ready-for-dev

## Story

As an operator, I want to trigger a deliberate `patronictl switchover` and confirm automatic promotion, so that FR10, FR11, and FR12 are verified and I have confidence in the HA topology.

## Acceptance Criteria

1. `patronictl switchover` completes without operator intervention
2. New primary is healthy and accepting connections within 30 seconds
3. `master.postgresql.service.consul` resolves to new primary after switchover
4. No data loss confirmed (row inserted before switchover is readable after)

## Tasks / Subtasks

- [ ] Pre-switchover verification: confirm cluster is healthy (AC: 1)
  - [ ] `patronictl -c /etc/patroni/patroni.yml list` shows Leader + Replica, both Running
- [ ] Insert test row before switchover (AC: 4)
  - [ ] `psql -h master.postgresql.service.consul -c "INSERT INTO failover_test VALUES (1)"`
  - [ ] Create `failover_test` table first if it doesn't exist
- [ ] Execute deliberate switchover (AC: 1)
  - [ ] `patronictl -c /etc/patroni/patroni.yml switchover postgres-{env} --master <current-primary> --candidate <replica>`
  - [ ] Command completes without requiring manual intervention
- [ ] Verify new primary state within 30 seconds (AC: 2)
  - [ ] `patronictl list` shows old replica now as Leader
  - [ ] `psql -h master.postgresql.service.consul -c "SELECT version()"` succeeds
- [ ] Verify Consul DNS routes to new primary (AC: 3)
  - [ ] `dig @consul-server master.postgresql.service.consul` returns new primary IP
- [ ] Verify no data loss (AC: 4)
  - [ ] `psql -h master.postgresql.service.consul -c "SELECT * FROM failover_test WHERE id=1"` returns row
- [ ] Document outcome in story completion notes

## Dev Notes

### Architecture Context

This is a verification story — it validates the Patroni cluster topology established in Story 2.2. All work is performed live against the running cluster. The story requires cluster nodes from EPIC-001 and Patroni cluster from Story 2.2 to be complete.

- Patroni scope: `postgres-{env}`
- Consul service: `master.postgresql.service.consul`
- `patronictl` operates via the Patroni REST API on each node
- After switchover, previous primary becomes replica; previous replica becomes primary

### Responsibility Split

**This story does:**
- Executes a deliberate controlled switchover
- Verifies automatic promotion, health, Consul DNS, and data integrity

**This story does NOT:**
- Test unplanned failure (node kill) — controlled switchover is sufficient evidence for homelab
- Implement any code changes — pure operational verification
- Modify any OpenTofu modules

### Directory Layout

```
terminus.infra/
  tests/
    acceptance/
      test-ha-failover.sh    # CREATE — scripted failover test with pass/fail output
```

### Testing Standards

- `patronictl switchover` exits 0
- Time from `switchover` command to new primary accepting connections: ≤ 30 seconds (manual stopwatch)
- `dig master.postgresql.service.consul` returns new primary IP (not old primary IP)
- Test row inserted before switchover is readable after switchover from new primary

### References

- epics.md: Story 2.5 Acceptance Criteria
- architecture.md: FR10 — Single node failure causes no data loss and no service interruption requiring operator intervention
- architecture.md: FR11 — Operator can verify cluster HA status at any time
- architecture.md: FR12 — Cluster failover is automatic; operator intervention not required

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tests/acceptance/test-ha-failover.sh` — CREATE (scripted HA verification)
