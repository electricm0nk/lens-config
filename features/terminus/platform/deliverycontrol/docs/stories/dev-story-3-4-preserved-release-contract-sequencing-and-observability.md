# Story 3.4: Preserved Release Contract Sequencing and Observability

Status: ready-for-dev

## Story

As a platform operator,
I want the new release path to preserve the established release activity sequence while improving observability,
so that automated releases remain understandable and debuggable without changing the core orchestration contract.

## Acceptance Criteria

1. `ReleaseWorkflow` still sequences `SeedSecrets`, optional `ProvisionDatabase`, `WaitForArgoCDSync`, and `RunSmokeTest`
2. The orchestration semantics remain consistent with the existing architecture
3. Operators can identify releases by branch, environment, SHA, and workflow ID across GitHub Actions, Temporal, and runbook guidance
4. Observability makes GitHub Actions visibly primary and Semaphore visibly fallback only
5. Failures remain attributable to a specific contract stage and the docs remain aligned with the implemented sequence

## Tasks / Subtasks

- [ ] Verify or update workflow sequencing so the established contract remains intact (AC: 1, 2)
- [ ] Add or align observability references across GitHub Actions output, workflow IDs, and docs (AC: 3, 4)
- [ ] Ensure failure attribution maps to specific contract stages (AC: 5)
- [ ] Update operational docs only where needed to remain aligned with the implemented release flow (AC: 5)

## Dev Notes

### Why This Story Matters

The initiative is explicitly not a replacement of `ReleaseWorkflow`. This story protects that promise while making the new entry path observable enough for operators to trust.

### Documentation Constraint

Keep docs synchronized with the implemented behavior. Drift between runbook and code would undermine the goal of a single clear release path.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L359)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L111)
- [docs/terminus/architecture.md](docs/terminus/architecture.md#L299)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List