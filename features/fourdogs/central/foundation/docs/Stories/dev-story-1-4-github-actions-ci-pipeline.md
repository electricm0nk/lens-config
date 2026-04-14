# Story 1.4: Semaphore CI Build Pipeline

Status: ready-for-dev

## Story

As Todd,
I want a Semaphore build pipeline that builds, tests, and publishes the container image on every push,
so that every merge to main produces a tested, deployable artifact pushed to GHCR from within the network.

> **Architecture constraint:** Do NOT use GitHub Actions or any external CI service. GHCR image builds and pushes must originate from within the `trantor.internal` network via SemaphoreUI. (See architecture.md — CI/CD Toolchain Decision Guide.)

## Acceptance Criteria

1. Pipeline stages in order: build → unit tests → `sqlc generate` diff check → Docker build → GHCR push
2. Stale `sqlc generate` output causes pipeline to fail before Docker build stage
3. Dockerfile uses multi-stage build: builder (Go toolchain) → final (`gcr.io/distroless/static-debian12`)
4. Final image contains both `api` and `migrate` binaries
5. Image pushed to `ghcr.io/electricm0nk/fourdogs-central` tagged with git SHA via Semaphore (not GitHub Actions)
6. All stages pass on clean repo with no test failures

## Tasks / Subtasks

- [ ] Task 0: Verify Story 1.3 complete — `sqlc generate` produces no diff

- [ ] Task 1: Create multi-stage Dockerfile (AC: 3, 4)
  - [ ] Create `Dockerfile` at repo root:
    ```dockerfile
    # Stage 1: builder
    FROM golang:1.22-bookworm AS builder
    WORKDIR /app
    COPY go.mod go.sum ./
    RUN go mod download
    COPY . .
    RUN CGO_ENABLED=0 GOOS=linux go build -o /out/api ./cmd/api
    RUN CGO_ENABLED=0 GOOS=linux go build -o /out/migrate ./cmd/migrate

    # Stage 2: final (distroless)
    FROM gcr.io/distroless/static-debian12
    COPY --from=builder /out/api /api
    COPY --from=builder /out/migrate /migrate
    ENTRYPOINT ["/api"]
    ```
  - [ ] Verify `gcr.io/distroless/static-debian12` is the correct tag (not `nonroot`)
  - [ ] Commit: `build(docker): add multi-stage Dockerfile with distroless final stage`

- [ ] Task 2: Create `scripts/ci.sh` — code quality gate (AC: 1, 2, 6)
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  echo "=== build ==="
  go build ./...

  echo "=== test ==="
  go test ./... -race -count=1

  echo "=== sqlc diff check ==="
  sqlc generate
  git diff --exit-code internal/db/ || {
    echo "ERROR: sqlc output is stale — run 'make generate' and commit"
    exit 1
  }
  ```
  - [ ] `chmod +x scripts/ci.sh`
  - [ ] Commit: `ci: add semaphore ci script`

- [ ] Task 3: Create `scripts/publish.sh` — Docker build + GHCR push (AC: 5)
  ```bash
  #!/usr/bin/env bash
  set -euo pipefail

  IMAGE="ghcr.io/electricm0nk/fourdogs-central"
  SHA="${GIT_SHA:-$(git rev-parse --short HEAD)}"

  echo "=== docker build ==="
  docker build -t "${IMAGE}:${SHA}" -t "${IMAGE}:latest" .

  echo "=== docker push ==="
  docker push "${IMAGE}:${SHA}"
  docker push "${IMAGE}:latest"

  echo "Published ${IMAGE}:${SHA}"
  ```
  - [ ] `chmod +x scripts/publish.sh`
  - [ ] Commit: `ci: add semaphore publish script for GHCR push`

- [ ] Task 4: Define Semaphore Task Templates in SemaphoreUI (AC: 1, 5)
  - [ ] Navigate to SemaphoreUI → Projects → New Project → `fourdogs-central`
    - Repository URL: `https://github.com/electricm0nk/fourdogs-central`
    - Branch: track all branches
  - [ ] Create Task Template **`fourdogs-central-ci`**:
    - Script: `bash scripts/ci.sh`
    - Environment vars: `SQLC_VERSION=latest`
    - Trigger: webhook on push to any branch
  - [ ] Create Task Template **`fourdogs-central-publish`**:
    - Script: `bash scripts/publish.sh`
    - Environment vars: `GIT_SHA={{ sha }}`
    - Trigger: webhook on push to `main` branch only
    - Requires: GHCR credentials available in Semaphore environment
  - [ ] Configure GitHub repository webhook to POST to the Semaphore webhook URL on push events

- [ ] Task 5: Test pipeline locally (AC: 6)
  - [ ] Run `docker build .` locally — ensure both binaries are present in the image
  - [ ] Run `docker run --rm fourdogs-central /migrate --help` — confirm binary exists
  - [ ] Run `docker run --rm fourdogs-central /api` — expect startup failure (missing env vars) but binary runs

- [ ] Task 6: Update sprint status
  - [ ] Set `1-4-semaphore-ci-pipeline: done`

## Dev Notes

- **No GitHub Actions** — GHCR pushes must originate from within the network. `scripts/publish.sh` runs in Semaphore on `trantor.internal`. See terminus domain architecture.
- `sqlc generate` in the CI script requires the schema (migrations SQL) to be present — no test DB needed for codegen only
- The diff check must run AFTER `sqlc generate` AND BEFORE the Docker build step — the script ordering enforces this
- `ghcr.io/electricm0nk/fourdogs-central` — image name must match the GitHub repository path exactly
- `gcr.io/distroless/static-debian12` — static distroless is correct for CGO_ENABLED=0 Go binaries
- GHCR credentials available as environment variables in SemaphoreUI — already seeded as part of infra setup
- `GIT_SHA` env var passed by Semaphore task runner at runtime; fallback to `git rev-parse` for local execution
- The migrate binary in the image is used as a k8s init container in the Helm chart (Story 1.5)

### Project Structure Notes

- Both `api` and `migrate` in the same image is simpler for the init container pattern
- `ENTRYPOINT ["/api"]` — the init container overrides this with `["/migrate", "up"]`

### References

- [Source: docs/terminus/architecture.md — CI/CD Toolchain Decision Guide]
- [Source: phases/techplan/architecture.md — CI Pipeline section, ARCH8]
- [Source: phases/devproposal/epics.md — Story 1.4 ACs]
- [Source: phases/businessplan/prd.md — FR26 (k3s deployment)]

## Dev Agent Record

### Agent Model Used

GitHub Copilot / Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List
