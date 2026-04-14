# Story 2.4: Safe Manifest Promotion for Dev Releases

Status: ready-for-dev

## Story

As a platform operator,
I want dev releases to update `terminus.infra` through a controlled commit on the self-hosted runner,
so that GitOps promotion is auditable, revertible, and safe under concurrent merge pressure.

## Acceptance Criteria

1. Dev release promotion updates only the dev values target for that service in `terminus.infra`
2. The promotion creates a release commit using the approved commit convention
3. Promotion logic uses explicit serialization or rebase-and-retry behavior to avoid conflicting writes
4. Promotion failure is surfaced as a hard workflow failure
5. Temporal is not invoked if manifest promotion fails

## Tasks / Subtasks

- [ ] Implement manifest update logic on the self-hosted runner for the dev values target only (AC: 1)
- [ ] Use the approved release commit message format for promotion commits (AC: 2)
- [ ] Add concurrency protection such as a branch-scoped concurrency group and/or rebase-retry logic (AC: 3)
- [ ] Ensure failed promotion exits the workflow as a hard failure (AC: 4)
- [ ] Gate Temporal start behind confirmed promotion success (AC: 5)

## Dev Notes

### Boundary Integrity

The architecture calls out the promotion-then-Temporal boundary as the primary consistency risk in the design. Treat manifest promotion as a first-class release step with explicit failure handling.

### Allowed Mechanism

This story intentionally uses a GitHub Actions step on the self-hosted runner rather than immediately moving promotion into Temporal. Do not overbuild durability here.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L272)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L173)
- [docs/terminus/platform/deliverycontrol/adversarial-review-report.md](docs/terminus/platform/deliverycontrol/adversarial-review-report.md#L26)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List