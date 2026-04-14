# Story 1.4: Release Gating and Audit Summary

Status: ready-for-dev

## Story

As a platform operator,
I want the workflow to stop cleanly before release execution when prerequisites fail,
so that the automated path remains auditable and does not create unsafe partial releases.

## Acceptance Criteria

1. When any prerequisite stage fails, manifest promotion and Temporal start are skipped
2. The workflow summary records the failed stage, branch, environment, and image context
3. GitHub Actions output clearly distinguishes build failure, promotion failure, and Temporal start failure
4. Audit output preserves a traceable release attempt even on failure
5. Break-glass guidance is presented as exceptional follow-up, not normal procedure

## Tasks / Subtasks

- [ ] Add explicit job gating so downstream release actions require prerequisite success (AC: 1)
- [ ] Write a workflow summary or equivalent audit output containing branch, environment, service, image, and failed stage (AC: 2, 4)
- [ ] Distinguish failure boundaries in operator-visible workflow output (AC: 3)
- [ ] Ensure fallback guidance points to the documented break-glass path without redefining the primary release path (AC: 5)
- [ ] Validate failure scenarios across build, runner handoff, and downstream release preparation (AC: 1, 3)

## Dev Notes

### Why This Story Exists

The architecture explicitly calls out the risk of split-brain behavior between manifest promotion and Temporal start. This story does not solve that entire boundary, but it creates the workflow gating and audit shape required to reason about failures safely.

### Required Audit Fields

At minimum, summary output should let an operator answer:

- what branch merged
- which environment was targeted
- which image was involved
- which stage failed
- whether Temporal was ever started

### Non-Goal

Do not build a separate release dashboard. GitHub Actions summary and disciplined step output are enough here.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L193)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L304)
- [docs/terminus/architecture.md](docs/terminus/architecture.md#L299)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List