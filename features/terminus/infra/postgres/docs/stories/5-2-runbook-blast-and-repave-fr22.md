# Story 5.2: Runbook: Blast-and-repave (FR22)

Status: ready-for-dev

## Story

As an operator, I want a runbook to reprovision the cluster from full VM destruction using only git and an age key, so that FR22 is satisfied and the procedure has been walked through at least once.

## Acceptance Criteria

1. Runbook covers: VM deletion → `sops -d` state decrypt → `tofu apply` reprovisioning → pgBackRest restore → acceptance test verification
2. Procedure executed end-to-end at least once; outcome recorded in runbook
3. Stored at `docs/terminus/infra/postgres/runbooks/02-blast-and-repave.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Pre-blast checklist — Confirm backup exists and is current; note last backup timestamp
  - [ ] Section 2: VM destruction — `tofu destroy -target module.postgres_cluster` (preserve backup VM)
  - [ ] Section 3: State decrypt — `sops -d terraform.tfstate.enc > terraform.tfstate`
  - [ ] Section 4: Reprovision — `tofu apply` (cluster VMs + packages + Patroni)
  - [ ] Section 5: pgBackRest restore — Stop PostgreSQL; `pgbackrest --stanza=main-dev restore`; Start Patroni
  - [ ] Section 6: Acceptance test — Run `tests/acceptance/run-acceptance-tests.sh` — all PASS
- [ ] Write each section with copy-pasteable commands (AC: 1, 2)
  - [ ] Include timing expectations at each step
  - [ ] Note: backup VM is NOT destroyed — this is the critical prerequisite
- [ ] Execute procedure end-to-end (AC: 2)
  - [ ] This story requires Stories 3.4 and 4.1 to be complete before end-to-end execution
  - [ ] Record execution outcome (elapsed time, any issues, resolution) in the runbook's "Validation Record" section
- [ ] Commit runbook with execution record to `terminus.infra` (AC: 3)

## Dev Notes

### Architecture Context

FR22 (blast-and-repave reprovisioning from git) requires the full round-trip: destroy VMs → reprovision from git → restore data from backup. The backup VM must survive the destroy step — the `tofu destroy` command must target only `module.postgres_cluster`, never `module.postgres_backup`.

- Critical constraint: `terraform.tfstate.enc` must be committed before VM destruction so state is recoverable
- Bootstrap order after repave: VMs provisioned → packages installed → Patroni started → pgBackRest config → restore
- Acceptance test after repave validates data integrity (acceptance_test_db rows intact)

### Responsibility Split

**This story does:**
- Authors the blast-and-repave runbook
- Executes it end-to-end at least once (coordination with Stories 3.4, 4.1 required)
- Records execution outcome

**This story does NOT:**
- Modify any infrastructure code
- Author other runbooks

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            02-blast-and-repave.md    # CREATE
```

### Testing Standards

- Runbook execution confirmed end-to-end (VM destroy → reprovision → restore → acceptance tests PASS)
- Elapsed time from VM destroy to acceptance test PASS recorded in runbook Validation Record section
- No manual VM changes outside the runbook steps
- Second operator can execute this runbook without asking the author any questions

### References

- epics.md: Story 5.2 Acceptance Criteria
- architecture.md: FR2 — Operator can reprovision cluster from git after full VM destruction with no manual steps beyond providing the age key
- architecture.md: FR22 — Blast-and-repave procedure executable end-to-end
- architecture.md: NFR7 — Blast-and-repave reproductibility from git

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/02-blast-and-repave.md` — CREATE
