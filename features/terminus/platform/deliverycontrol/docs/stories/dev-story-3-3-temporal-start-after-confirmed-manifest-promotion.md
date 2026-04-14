# Story 3.3: Temporal Start After Confirmed Manifest Promotion

Status: ready-for-dev

## Story

As a platform operator,
I want GitHub Actions to start `ReleaseWorkflow` only after manifest promotion is confirmed pushed,
so that Temporal always observes a release candidate with a corresponding GitOps commit to wait on.

## Acceptance Criteria

1. After confirmed manifest promotion, GitHub Actions invokes `ReleaseWorkflow` with `ServiceName`, `ProvisionDB`, `Environment`, and `ImageTag`
2. The invocation uses the environment-aware workflow ID and duplicate-failed-only reuse policy
3. `temporal workflow start` runs from the self-hosted runner through the approved internal access method
4. The flow does not depend on cloud-runner network reachability
5. If Temporal start fails after a successful promotion commit, the workflow is marked failed and correlates the failure to the promoted commit and workflow ID

## Tasks / Subtasks

- [ ] Gate Temporal invocation on confirmed manifest push success (AC: 1)
- [ ] Invoke `ReleaseWorkflow` with the extended input contract and environment-aware workflow ID (AC: 1, 2)
- [ ] Implement the approved self-hosted runner execution path for Temporal CLI access (AC: 3, 4)
- [ ] Surface post-promotion Temporal start failures clearly in workflow output (AC: 5)
- [ ] Preserve correlation between promotion commit and workflow ID for debugging (AC: 5)

## Dev Notes

### Boundary Rule

This story sits directly on the highest-risk boundary called out in the architecture: manifest promotion and Temporal start must remain coupled in behavior and observable in failure.

### Approved Access Pattern

The architecture’s preferred invocation path is from the self-hosted runner through cluster-local access to Temporal tooling. Do not move this back to a cloud runner.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L342)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L251)
- [docs/cicd.md](docs/cicd.md#L47)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List