# Story 2.3: Environment-Specific Ingress, DNS, and Smoke-Test Assets

Status: ready-for-dev

## Story

As a platform operator,
I want each environment to have its own ingress, DNS, and smoke-test path,
so that release verification targets the correct live endpoint for dev and prod.

## Acceptance Criteria

1. Dev ingress uses `{service}-dev.trantor.internal` and namespace `{service}-dev`
2. Prod ingress remains `{service}.trantor.internal` and namespace `{service}`
3. Required dev DNS records and supporting operator notes exist
4. Verification paths use external DNS hosts rather than cluster-internal service addresses
5. Both `verify-{service}-dev-deploy` and `verify-{service}-deploy` exist where required

## Tasks / Subtasks

- [ ] Add or update environment-specific ingress manifests for dev and prod naming (AC: 1, 2)
- [ ] Document or provision required dev DNS records through the approved operator path (AC: 3)
- [ ] Ensure smoke-test design points at external DNS hostnames, not `svc.cluster.local` addresses (AC: 4)
- [ ] Register or update both environment-specific verify templates in Semaphore where the service requires them (AC: 5)
- [ ] Validate that environment naming matches the canonical contract from Story 2.1 (AC: 1, 2, 5)

## Dev Notes

### Smoke-Test Rule

The runbook is explicit: verification playbooks must use the external `.trantor.internal` hostnames and should not depend on cluster-internal service discovery for release validation.

### Cross-System Nature

This story spans ingress manifests, DNS operations, and Semaphore templates. Keep the names uniform across all three systems.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L255)
- [docs/cicd.md](docs/cicd.md#L123)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L282)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List