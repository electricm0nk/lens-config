# Story 2.1: Canonical Environment Mapping Contract

Status: ready-for-dev

## Story

As a platform maintainer,
I want a single canonical mapping for branch, environment, namespace, values file, ArgoCD app, and smoke-test naming,
so that release automation and infra assets do not drift across services.

## Acceptance Criteria

1. One authoritative mapping defines `develop -> dev` and `main -> prod`
2. The same mapping defines namespace, values file, ArgoCD app, ingress host, and smoke-test naming for each environment
3. Downstream workflow and infra stories source names from this mapping instead of duplicating them
4. New services adopting deliverycontrol use the same mapping without local exceptions

## Tasks / Subtasks

- [ ] Choose and implement the single source of truth for environment mapping used by workflow and infra logic (AC: 1, 2)
- [ ] Define all naming outputs required by later stories: namespace, values file, ArgoCD app, ingress host, smoke-test template (AC: 2)
- [ ] Refactor or structure downstream usage so release automation reads from the mapping instead of re-deriving names ad hoc (AC: 3)
- [ ] Document how new services adopt the mapping contract (AC: 4)

## Dev Notes

### Important Execution Note

The readiness assessment flagged this as a must-resolve implementation detail early in Epic 2. Pick the concrete artifact here and use it consistently afterward.

### Acceptable Shapes

The architecture does not force a single technical implementation, but whatever you choose must be shared, reviewable, and hard to drift from. Examples: reusable workflow variables, checked-in config, or a shared helper script.

### Required Naming Outputs

The mapping must at least cover:

- branch -> environment
- environment -> namespace
- environment -> values file
- environment -> ArgoCD application name
- environment -> ingress host
- environment -> smoke-test template name

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L221)
- [docs/terminus/platform/deliverycontrol/implementation-readiness.md](docs/terminus/platform/deliverycontrol/implementation-readiness.md#L73)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L182)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List