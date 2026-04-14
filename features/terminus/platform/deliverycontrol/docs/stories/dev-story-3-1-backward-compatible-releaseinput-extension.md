# Story 3.1: Backward-Compatible ReleaseInput Extension

Status: ready-for-dev

## Story

As a platform developer,
I want `ReleaseInput` extended with environment-aware fields,
so that the existing Temporal release contract can distinguish dev and prod while still accepting legacy manual starts.

## Acceptance Criteria

1. `ReleaseInput` includes `Environment` and `ImageTag` in addition to existing fields
2. Legacy manual invocations that omit the new fields remain compatible with safe default behavior
3. Tests cover both legacy and new payload shapes
4. Workflow execution can distinguish explicit `dev` and `prod` contexts
5. Environment and image tag are visible for audit purposes without introducing secrets into workflow input

## Tasks / Subtasks

- [ ] Extend `ReleaseInput` with `Environment` and `ImageTag` while preserving existing fields (AC: 1)
- [ ] Implement defaulting or backward-compatibility handling for older manual starts (AC: 2)
- [ ] Add serialization and defaulting tests for legacy and new payload shapes (AC: 3)
- [ ] Verify environment-aware execution semantics in workflow-level tests (AC: 4)
- [ ] Ensure the added fields improve auditability without carrying secrets (AC: 5)

## Dev Notes

### Existing Contract

The current `ReleaseWorkflow` already exists in `terminus.platform`. This story extends its input contract rather than replacing the workflow or introducing a separate input type.

### Backward Compatibility

Existing Semaphore-started manual runs must continue to work. Default behavior should remain safe and explicit; the architecture notes `dev` as the default when absent.

### Audit-Only Nature of ImageTag

`ImageTag` is advisory and audit-focused. The workflow should not start pulling or resolving images from this field directly.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L308)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L220)
- [docs/terminus/platform/releaseorchestrator/architecture.md](docs/terminus/platform/releaseorchestrator/architecture.md#L78)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List