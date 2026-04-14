# Story 2.2: Dev GitOps Assets for a Service

Status: ready-for-dev

## Story

As a service developer,
I want dev values and ArgoCD applications created alongside prod assets,
so that merged code can be promoted into a distinct dev environment without disturbing production.

## Acceptance Criteria

1. `values-dev.yaml` exists for the service with dev-specific image override behavior
2. Dev ArgoCD application definitions target the `develop` branch and `{service}-dev` namespace
3. Dev apps use the dev values file and dev namespace
4. Prod apps remain on their production branch and namespace
5. The service can be synced into dev independently of prod

## Tasks / Subtasks

- [ ] Add `values-dev.yaml` to the service Helm values path in `terminus.infra` (AC: 1)
- [ ] Create dev ArgoCD application manifests that target `develop` and `{service}-dev` (AC: 2)
- [ ] Ensure the dev application uses the dev values file and namespace (AC: 3)
- [ ] Preserve existing prod app targeting and configuration (AC: 4)
- [ ] Validate independent dev sync behavior without impacting prod (AC: 5)

## Dev Notes

### Repo Boundary

This story primarily lands in `terminus.infra`, not the service repo. The release-control design deliberately separates service build logic from GitOps deployment state.

### Minimum Dev Values Content

The CI/CD runbook defines `values-dev.yaml` as a minimal override with image tag focus. Do not duplicate full production configuration unnecessarily unless the existing chart structure demands it.

### ArgoCD Pattern

Dev app definitions should mirror the production shape closely enough that environment drift stays low, with branch, namespace, and value-file differences explicit.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L238)
- [docs/cicd.md](docs/cicd.md#L84)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L261)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List