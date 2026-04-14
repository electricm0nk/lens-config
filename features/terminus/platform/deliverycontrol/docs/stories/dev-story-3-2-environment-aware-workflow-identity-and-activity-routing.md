# Story 3.2: Environment-Aware Workflow Identity and Activity Routing

Status: ready-for-dev

## Story

As a platform developer,
I want workflow identity and activity routing to include environment context,
so that dev and prod releases for the same SHA remain distinct and execute the correct verification path.

## Acceptance Criteria

1. Workflow IDs follow `release-{service}-{env}-{sha[:8]}`
2. The ID policy prevents duplicate successful releases while permitting reruns after failure
3. Dev releases route to `verify-{service}-dev-deploy` and prod releases route to `verify-{service}-deploy`
4. Secret seeding remains compatible with shared seeding conventions
5. Tests cover dev/prod template selection, workflow ID formatting, and optional database behavior

## Tasks / Subtasks

- [ ] Implement environment-aware workflow ID generation using the approved format (AC: 1)
- [ ] Preserve the duplicate-failed-only reuse policy in the updated workflow invocation path (AC: 2)
- [ ] Route smoke-test template selection by environment (AC: 3)
- [ ] Preserve existing secret-seeding compatibility while extending routing logic (AC: 4)
- [ ] Add tests for workflow ID formatting, template selection, and optional DB branching (AC: 5)

## Dev Notes

### Collision Prevention

This story exists because the same SHA can legitimately move through dev and prod. Environment must therefore be part of workflow identity, not just logging.

### Routing Boundary

The architecture keeps secret seeding relatively stable but makes smoke-test routing environment-aware. Avoid overgeneralizing the routing change beyond what the contract requires.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L325)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L232)
- [docs/cicd.md](docs/cicd.md#L56)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List