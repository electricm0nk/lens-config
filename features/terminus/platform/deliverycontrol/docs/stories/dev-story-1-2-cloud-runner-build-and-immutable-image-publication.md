# Story 1.2: Cloud Runner Build and Immutable Image Publication

Status: ready-for-dev

## Story

As a service developer,
I want the workflow to build and publish an immutable SHA-tagged image on the cloud runner path,
so that each release candidate is traceable to a specific commit and suitable for promotion.

## Acceptance Criteria

1. The cloud-runner job builds the service image on merged PRs targeting `develop`
2. The job pushes `ghcr.io/{org}/{service}:{sha}` to GHCR
3. The full pushed image tag is emitted as a downstream job output
4. The normal release path does not depend on `latest`
5. If image build or push fails, the workflow fails before any internal release step begins

## Tasks / Subtasks

- [ ] Add a `build-and-push` job to `.github/workflows/release.yml` that runs on a cloud runner (AC: 1)
- [ ] Derive the immutable image tag from the merge SHA and publish to GHCR (AC: 2)
- [ ] Emit the full image reference as a job output for downstream release jobs (AC: 3)
- [ ] Ensure the workflow treats the SHA tag as the artifact of record and does not rely on `latest` (AC: 4)
- [ ] Validate that build or push failure prevents manifest promotion or Temporal start (AC: 5)

## Dev Notes

### Scope Boundary

This story owns only the cloud-side image production path. Do not implement manifest promotion, self-hosted runner logic, or Temporal start here.

### Build Contract

The architecture locks the image strategy to immutable SHA tags:

- `ghcr.io/{org}/{service}:{sha}`
- `latest` forbidden for the normal release path

The output of this story is not just an image in GHCR; it is a reliable artifact reference that later stories consume.

### Runner Separation

This job must remain on a GitHub-hosted cloud runner. Internal-network access belongs to the self-hosted k3s runner in Story 1.3.

### Failure Boundary

If build or push fails, the workflow must stop before any internal promotion behavior. That boundary is a key part of the release contract.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L161)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L169)
- [docs/cicd.md](docs/cicd.md#L37)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List