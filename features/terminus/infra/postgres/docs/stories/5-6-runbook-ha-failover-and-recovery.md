# Story 5.6: Runbook: HA failover and recovery

Status: ready-for-dev

## Story

As an operator, I want a runbook for manual switchover, unplanned failover response, and re-joining a recovered node, so that any HA event can be handled without prior context.

## Acceptance Criteria

1. Covers: `patronictl switchover` → health verification → re-joining replica after repair
2. Stored at `docs/terminus/infra/postgres/runbooks/06-ha-failover.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Verify cluster health before failover — `patronictl list`
  - [ ] Section 2: Deliberate switchover — `patronictl switchover postgres-dev --master <primary> --candidate <replica>`
  - [ ] Section 3: Post-switchover health check — Confirm new primary, Consul DNS update, test query
  - [ ] Section 4: Unplanned failover response — What to observe after node loss; confirm auto-promotion occurred; confirm Consul DNS updated
  - [ ] Section 5: Re-joining recovered node — Start Patroni on recovered node; `patronictl list` shows it re-joins as replica
  - [ ] Section 6: Split-brain prevention — How to confirm no split-brain; `patronictl topology`
- [ ] Write each section with copy-pasteable commands (AC: 1)
  - [ ] Include expected `patronictl list` output at each verification step
  - [ ] Include timing expectations (e.g., replica re-sync time)
- [ ] Commit runbook to `terminus.infra` (AC: 2)

## Dev Notes

### Architecture Context

Patroni handles automatic failover without operator intervention (FR12). This runbook documents both the deliberate path (manual switchover for maintenance) and the response path (operator awareness when auto-failover occurs). Split-brain is prevented by Consul DCS quorum.

- `patronictl` CLI: installed on automation server or cluster nodes
- Consul DNS: `master.postgresql.service.consul` → current primary
- Recovery: failed node restarts Patroni and re-syncs as replica automatically
- Split-brain check: `patronictl topology` or check Consul healthchecks

### Responsibility Split

**This story does:**
- Authors the HA failover and recovery runbook

**This story does NOT:**
- Execute a new failover test (Story 2.5 already verified this)
- Modify any Patroni or Consul configuration

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            06-ha-failover.md    # CREATE
```

### Testing Standards

- Runbook reviewed against actual `patronictl` output from Story 2.5 switchover execution
- All three paths covered: deliberate switchover, unplanned failover response, node re-join
- Commands copy-pasteable with no ambiguous variable substitutions

### References

- epics.md: Story 5.6 Acceptance Criteria
- architecture.md: FR10 — HA single node failure; FR11 — HA status verifiable; FR12 — Automatic failover
- architecture.md: "Consul service master.postgresql.service.consul → primary VM"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/06-ha-failover.md` — CREATE
