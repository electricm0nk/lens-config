# Story 2.5: Production Promotion Reuses the Dev-Validated Artifact

Status: ready-for-dev

## Story

As a service developer,
I want the production promotion path to reuse the artifact that already passed dev,
so that production deploys exactly the image that was validated earlier and does not rebuild a different binary.

## Acceptance Criteria

1. The production path resolves the previously built SHA artifact for the merged code
2. The production path does not rebuild a new image
3. The prod values target is updated with the same SHA validated in dev
4. The resulting commit remains traceable to the original artifact
5. If the required artifact cannot be resolved or is absent in GHCR, the workflow fails before manifest promotion or Temporal start

## Tasks / Subtasks

- [ ] Choose and document the source of truth for artifact reuse resolution before coding the production path (AC: 1)
- [ ] Implement artifact lookup for the merged code path without rebuilding (AC: 1, 2)
- [ ] Update the prod values target with the previously validated SHA (AC: 3)
- [ ] Preserve audit traceability from production promotion back to the original dev-validated artifact (AC: 4)
- [ ] Hard-fail the workflow if the artifact cannot be resolved or does not exist (AC: 5)

## Dev Notes

### Readiness Note

The implementation-readiness review flagged this story as needing an explicit artifact source of truth. Resolve that design choice first and keep it documented in the implementation record.

### Non-Negotiable Rule

No rebuild on `main`. The production path must promote the same artifact that passed dev verification.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L289)
- [docs/terminus/platform/deliverycontrol/implementation-readiness.md](docs/terminus/platform/deliverycontrol/implementation-readiness.md#L83)
- [docs/cicd.md](docs/cicd.md#L41)