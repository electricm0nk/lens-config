# Story 1.3: Go Dockerfile and CI Pipeline Update

Status: ready-for-dev

## Story

As a platform developer,
I want a Go multi-stage Dockerfile and updated CI pipeline so that the worker image builds from Go source and is pushed to GHCR under the same image name,
so that the existing Helm chart and ArgoCD deployment work without modification.

## Acceptance Criteria

1. `Dockerfile` at repo root is a multi-stage Go build: stage 1 uses `golang:1.25` to build the binary, stage 2 copies binary into `gcr.io/distroless/static:nonroot`
2. The final image runs as non-root (distroless nonroot enforces this)
3. Image still tagged and pushed to `ghcr.io/electricm0nk/terminus-platform-worker` (same registry path)
4. `helm/temporal-worker/values.yaml` is unchanged — `image.repository` and `image.tag` are the same
5. The CI pipeline Dockerfile path reference is updated to point to the new `Dockerfile` at repo root
6. `docker build .` completes locally without errors
7. The built image entrypoint is the Go binary (not `node services/temporal/dist/worker.js`)

**Pre-condition:** Story 1.2 is complete and `go build ./cmd/worker/` passes.

## Tasks / Subtasks

- [ ] Write `Dockerfile` at repo root — Go multi-stage (AC: 1, 2, 7)
  - [ ] Stage 1: `FROM golang:1.25 AS builder` → `go build -o /worker ./cmd/worker/`
  - [ ] Stage 2: `FROM gcr.io/distroless/static:nonroot` → `COPY --from=builder /worker /worker` → `ENTRYPOINT ["/worker"]`
- [ ] Find CI pipeline config — identify the current Dockerfile path reference (AC: 5)
- [ ] Update CI pipeline Dockerfile path from `services/temporal/Dockerfile` to `./Dockerfile` (AC: 5)
- [ ] Verify `docker build .` succeeds locally (AC: 6)
- [ ] Verify image name and tag in CI matches `ghcr.io/electricm0nk/terminus-platform-worker` (AC: 3)
- [ ] Confirm `helm/temporal-worker/values.yaml` is unchanged (AC: 4)

## Dev Notes

### Dockerfile

Create at repo root (not in `services/temporal/`):

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.25 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /worker ./cmd/worker/

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /worker /worker
ENTRYPOINT ["/worker"]
```

**Notes:**
- `CGO_ENABLED=0` — required for distroless static runtime (no libc)
- `GOOS=linux` — explicit cross-compile target for Linux
- `/worker` as entrypoint — consistent with name used in `go build -o /worker`
- `distroless/static:nonroot` — runs as UID 65532, no shell, minimal attack surface

### CI Pipeline

The CI pipeline config is likely at `.github/workflows/` (GitHub Actions) or equivalent. Find the file that:
- References `services/temporal/Dockerfile`
- Pushes to `ghcr.io/electricm0nk/terminus-platform-worker`

Update only the `--file` or `context` reference for the Dockerfile path. Do not change image names, tags, or push targets. Example in GitHub Actions:

```yaml
# Before:
- uses: docker/build-push-action@v5
  with:
    context: .
    file: services/temporal/Dockerfile

# After:
- uses: docker/build-push-action@v5
  with:
    context: .
    file: Dockerfile
```

### Image Tag and Registry

The image name `ghcr.io/electricm0nk/terminus-platform-worker` must remain unchanged — the Helm chart and ArgoCD Application reference it. Check `helm/temporal-worker/values.yaml` to confirm the current value before and after this change.

### Helm Chart Unchanged

The PR for this story must NOT modify `helm/temporal-worker/values.yaml`, `helm/temporal-worker/Chart.yaml`, or any k8s manifest. The only changed files are `Dockerfile` (new) and the CI pipeline config.

### Old Dockerfile

`services/temporal/Dockerfile` (Node.js multi-stage) still exists in this story — it is removed in Story 1.4 with the rest of the TypeScript scaffold. Do not delete it here.

### Verification

Before submitting this story:
```bash
docker build -t terminus-platform-worker:test .
docker run --rm terminus-platform-worker:test  # Should exit with connection error (no Temporal server) — NOT "command not found"
```

If the entrypoint is wrong, `docker run` will exit with a different error message.

### References

- [Architecture: Infrastructure & Deployment](docs/terminus/platform/releaseorchestrator/architecture.md#infrastructure--deployment)
- [Architecture: Project Structure](docs/terminus/platform/releaseorchestrator/architecture.md#repository-transformation)
- [Epics: Story 1.3](docs/terminus/platform/releaseorchestrator/epics.md#story-13-go-dockerfile-and-ci-pipeline-update)
- Adversarial review Finding 3 (migration atomicity): Story 1.4 MUST NOT be merged before this story

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
