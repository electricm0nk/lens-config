# Story 4.3: Break-Glass Release Path and Operator Guardrails

Status: ready-for-dev

## Story

As a platform operator,
I want a documented manual release fallback that mirrors the normal release contract,
so that releases can proceed during GitHub Actions outages without normalizing manual invocation.

## Acceptance Criteria

1. When GitHub Actions is unavailable, Semaphore can start the same `ReleaseWorkflow` contract with the environment-aware input shape
2. The manual fallback still runs through the same downstream release stages
3. Runbook and supporting docs say clearly that the path is exceptional and not the default
4. A manual fallback run remains traceable by environment, SHA, and workflow ID
5. The fallback does not create a separate incompatible release contract

## Tasks / Subtasks

- [ ] Update the manual fallback path so Semaphore can start the environment-aware `ReleaseWorkflow` input shape (AC: 1)
- [ ] Verify the fallback preserves the same downstream activity sequence as the normal path (AC: 2, 5)
- [ ] Document the fallback as exceptional and subordinate to GitHub Actions in runbook guidance (AC: 3)
- [ ] Preserve traceability fields for manual fallback runs: environment, SHA, workflow ID (AC: 4)

## Dev Notes

### Guardrail, Not Alternate System

This story is not permission to maintain two equal release systems. The normal path remains GitHub Actions. Semaphore exists only for outages and operator recovery.

### Compatibility Requirement

The fallback must use the same evolved release contract, including environment-aware input shape and workflow identity.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L431)
- [docs/cicd.md](docs/cicd.md#L60)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L147)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List