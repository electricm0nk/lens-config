# Story 1.3: Self-Hosted Runner Release Handoff

Status: ready-for-dev

## Story

As a platform operator,
I want internal release steps to execute on a self-hosted k3s runner,
so that manifest promotion and Temporal invocation can reach `*.trantor.internal` services safely.

## Acceptance Criteria

1. The post-build release job runs on `[self-hosted, trantor-internal]`
2. The internal release job receives environment, service, and image outputs from earlier jobs
3. The internal release job uses those outputs as the single source of truth for downstream promotion logic
4. If the self-hosted runner is unavailable or misconfigured, the workflow remains failed and visible in GitHub Actions
5. No operator-facing text suggests bypassing the normal path when the self-hosted runner is unavailable

## Tasks / Subtasks

- [ ] Add a self-hosted release job to `release.yml` with `runs-on: [self-hosted, trantor-internal]` (AC: 1)
- [ ] Wire job dependencies so the internal job consumes outputs from the cloud build stage (AC: 2)
- [ ] Normalize downstream environment, service, and image inputs so later stories do not re-derive them inconsistently (AC: 3)
- [ ] Validate failure behavior when the runner cannot be scheduled or does not have the required labels (AC: 4)
- [ ] Keep workflow messaging aligned with GitHub Actions as primary and Semaphore as break-glass only (AC: 5)

## Dev Notes

### Core Constraint

Any step that needs access to `*.trantor.internal` must run on the self-hosted runner. This is a hard architectural rule, not an optimization.

### Scope Boundary

This story establishes the handoff environment only. Do not implement manifest commits or Temporal invocation here; those belong to later stories.

### Output Discipline

Use outputs from the earlier jobs rather than recomputing branch, environment, or image details. This is part of the anti-drift requirement across the release pipeline.

### Failure Behavior

Runner scheduling failure is a release failure. Do not silently fall back to cloud execution or reinterpret Semaphore as the normal path.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L177)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L191)
- [docs/cicd.md](docs/cicd.md#L13)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List