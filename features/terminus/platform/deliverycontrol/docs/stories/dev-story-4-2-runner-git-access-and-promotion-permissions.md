# Story 4.2: Runner Git Access and Promotion Permissions

Status: ready-for-dev

## Story

As a platform operator,
I want the self-hosted runner to have the minimum git access needed to promote manifests,
so that it can push controlled release commits to `terminus.infra` without broad unmanaged repository privilege.

## Acceptance Criteria

1. Repository credentials are delivered through approved secret-management paths
2. Credentials are scoped to the repositories and actions needed for release promotion
3. Promotion commits are attributable to the automation path through their git identity
4. If repository access is missing or insufficient, the workflow fails before Temporal start
5. Missing permission surfaces as an operational setup defect, not a silent fallback

## Tasks / Subtasks

- [ ] Provision automation git credentials through the approved secret path (AC: 1)
- [ ] Scope credentials narrowly to `terminus.infra` promotion behavior and required repositories (AC: 2)
- [ ] Set a traceable automation git identity for promotion commits (AC: 3)
- [ ] Ensure permission failure prevents downstream Temporal invocation (AC: 4)
- [ ] Surface missing-permission errors clearly in workflow output and runbook guidance (AC: 5)

## Dev Notes

### Readiness Note

The readiness assessment explicitly called out credential scoping as an execution guardrail. Keep the permission surface narrow and documented.

### Failure Boundary

Lack of repo access is a setup defect. The workflow must stop before Temporal start and report that exact cause.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L411)
- [docs/terminus/platform/deliverycontrol/implementation-readiness.md](docs/terminus/platform/deliverycontrol/implementation-readiness.md#L90)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L353)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List