# Story 5.9: SoD deviation doc

Status: ready-for-dev

## Story

As an operator, I want the single-operator SoD deviation formally documented, so that the A6 adversarial finding is closed and future auditors have a record of the deliberate deviation.

## Acceptance Criteria

1. `docs/terminus/infra/postgres/deviations/sod-deviation.md` created
2. Documents: what SoD requires, what the homelab reality is, who approved the deviation, and mitigating controls in place

## Tasks / Subtasks

- [ ] Create `docs/terminus/infra/postgres/deviations/` directory in `terminus.infra` (AC: 1)
- [ ] Write SoD deviation document (AC: 2)
  - [ ] Section 1: What SoD requires — Separation between the operator who provisions infrastructure and the operator who has access to production data
  - [ ] Section 2: Homelab reality — Single operator (Todd) performs both roles; no second operator available in this environment
  - [ ] Section 3: Adversarial finding — Reference A6 adversarial review finding from TechPlan adversarial review
  - [ ] Section 4: Who approved — Approver: CrisWeber; date: (date of stakeholder approval)
  - [ ] Section 5: Mitigating controls — List all compensating controls:
    - [ ] All access logged via pgaudit (NFR12)
    - [ ] All provisioning via OpenTofu (no manual VM changes outside documented break-glass)
    - [ ] Break-glass credential isolated in Vault and rotated post-use
    - [ ] SOPS-encrypted state prevents unauthorized reprovisioning
    - [ ] All runbooks documented — any other operator can reproduce or audit every action
- [ ] Commit deviation doc to `terminus.infra` (AC: 1)

## Dev Notes

### Architecture Context

The adversarial review (TechPlan phase) flagged as A6 that a single operator performing both infrastructure provisioning and direct data access violates SoD principles. The deviation was deferred to implementation phase for formal documentation. CrisWeber provided stakeholder approval of the overall initiative including awareness of this deviation.

- File path: `docs/terminus/infra/postgres/deviations/sod-deviation.md`
  - Note: the architecture doc references `docs/terminus/infra/postgres/deviations/sod-deviation.md` as the target path within `terminus.infra`
- Approver: CrisWeber (stakeholder approval recorded in `docs/terminus/infra/postgres/stakeholder-approval.md`)
- Deferred from: TechPlan adversarial review finding A6

### Responsibility Split

**This story does:**
- Authors the SoD deviation document

**This story does NOT:**
- Change any controls or architecture
- Address other adversarial findings (those were addressed in the implementation-readiness phase)

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          deviations/
            sod-deviation.md    # CREATE
```

### Testing Standards

- Document contains all 5 required sections (SoD requirement, homelab reality, adversarial finding reference, approver, mitigating controls)
- Approver field references CrisWeber with approval date
- All 5 mitigating controls listed
- Document is readable by a future auditor with no prior context

### References

- epics.md: Story 5.9 Acceptance Criteria
- architecture.md: "SoD deviation doc: docs/terminus/infra/postgres/deviations/sod-deviation.md (deferred from TechPlan)"
- docs/terminus/infra/postgres/stakeholder-approval.md — approver and approval date reference

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/deviations/sod-deviation.md` — CREATE
