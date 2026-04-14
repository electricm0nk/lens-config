# Story 3.5: Temporal-Controlled ArgoCD Sync After Release Prerequisites

Status: ready-for-dev

## Story

As a platform operator,
I want `ReleaseWorkflow` to trigger ArgoCD sync only after secrets and database prerequisites are complete,
so that release-managed apps do not begin deployment before Temporal has finished the steps that make the deployment safe to start.

## Acceptance Criteria

1. After manifest promotion and Temporal start, `ReleaseWorkflow` explicitly triggers sync for the environment-specific ArgoCD app only after `SeedSecrets` and optional `ProvisionDatabase` succeed
2. `WaitForArgoCDSync` executes only after that explicit sync request and continues to observe the same environment-specific ArgoCD app until it is `Healthy` and `Synced`
3. Release-managed ArgoCD applications do not roll workloads forward before the Temporal-controlled sync step, while Git remains the source of desired state
4. If the explicit ArgoCD sync request fails or is rejected, the workflow stops before smoke testing and surfaces the failure as a distinct sync-trigger boundary
5. The revised flow remains backward compatible with the existing release contract and break-glass entry path

## Tasks / Subtasks

- [ ] Add a Temporal activity or equivalent workflow step that requests sync for the environment-specific ArgoCD application after prerequisites complete (AC: 1)
- [ ] Ensure `WaitForArgoCDSync` runs only after the explicit sync step and targets the same app identity (AC: 2)
- [ ] Adjust release-managed ArgoCD sync policy or introduce an equivalent gate so deployment does not start ahead of Temporal's sync decision (AC: 3)
- [ ] Surface sync-trigger failures distinctly in Temporal logs, GitHub Actions output, and runbook guidance (AC: 4)
- [ ] Verify manual or break-glass starts continue to honor the same release contract without reintroducing premature deployment (AC: 5)

## Dev Notes

### Contract Boundary

This story changes who controls deployment timing, not who owns desired state. Git still declares the image tag and ArgoCD remains the only delivery engine. Temporal simply becomes responsible for deciding when reconciliation starts for release-managed apps.

### Preferred Shape

Favor a dedicated sync-trigger step in `ReleaseWorkflow` over hidden timing assumptions. The release should read as: promote manifest, start Temporal, seed secrets, provision database if needed, trigger ArgoCD sync, wait for sync health, run smoke test.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L405)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L20)
- [docs/cicd.md](docs/cicd.md#L324)
- [TargetProjects/terminus/platform/terminus.platform/internal/activities/release.go](TargetProjects/terminus/platform/terminus.platform/internal/activities/release.go#L91)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List