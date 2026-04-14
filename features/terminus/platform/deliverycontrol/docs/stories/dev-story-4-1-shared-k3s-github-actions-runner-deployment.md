# Story 4.1: Shared k3s GitHub Actions Runner Deployment

Status: ready-for-dev

## Story

As a platform operator,
I want a shared self-hosted GitHub Actions runner deployed in k3s,
so that internal release jobs have a governed execution environment with cluster-reachable network access.

## Acceptance Criteria

1. The runner deployment in `terminus.infra` includes deployment, RBAC, and secret delivery assets needed for registration and execution
2. The runner advertises the labels required by the release workflow
3. Runner registration credentials follow the Vault KV to ESO to k8s Secret pattern
4. No static registration token is committed to git
5. The runner appears as an ArgoCD-managed workload in k3s with its purpose documented

## Tasks / Subtasks

- [ ] Add deployment, RBAC, and secret delivery assets for the shared runner in `terminus.infra` (AC: 1)
- [ ] Configure the runner with the labels required by the release workflow (AC: 2)
- [ ] Deliver registration credentials through Vault and ESO (AC: 3, 4)
- [ ] Ensure the runner is deployed through ArgoCD and documented as internal release infrastructure (AC: 5)

## Dev Notes

### Infrastructure Location

The architecture places this runner in k3s as an infra concern, not inside individual service repos. Keep that boundary intact.

### Token Handling

Do not commit registration credentials or leave them in plain k8s secrets without the established Vault/ESO path.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L390)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L206)
- [docs/cicd.md](docs/cicd.md#L18)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List