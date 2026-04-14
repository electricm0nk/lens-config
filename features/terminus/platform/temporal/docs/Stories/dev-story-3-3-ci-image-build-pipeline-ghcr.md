# Story 3.3: CI Image Build Pipeline — GHCR

Status: done

## Story

As a platform developer,
I want a GitHub Actions CI pipeline that builds the TypeScript worker Docker image and pushes it to GHCR on every merge to `main`,
so that the worker Deployment in k3s always reflects the latest committed worker source code.

## Acceptance Criteria

1. `.github/workflows/publish-images.yml` fully implemented:
   - Trigger: `push` to `main` branch, `paths: ["services/temporal/**"]`
   - Build: `docker build -f services/temporal/Dockerfile -t ghcr.io/electricm0nk/terminus-platform-worker:${{ github.sha }} .`
   - Push: auth to GHCR via `GITHUB_TOKEN`, push both `:{sha}` and `:latest` tags
   - On success: output image digest
2. `services/temporal/Dockerfile` created:
   - Multi-stage: build stage compiles TypeScript; runtime stage runs compiled JS
   - Base image: `node:lts-alpine` (LTS + alpine for minimal size)
   - Installs only production dependencies in runtime stage (`npm ci --omit=dev`)
   - Non-root user in runtime stage
   - Entrypoint: `node dist/worker.js`
3. CI workflow runs successfully on merge to `main` and pushes image to GHCR
4. Worker Deployment in Story 3.1 updated to reference image digest (replace bootstrap placeholder)
5. **N7 bootstrap sequence:** After CI publishes first real image, `helm/temporal-worker/values.yaml` `image.tag` pinned to first successful build digest. This one-time step documented in story completion notes.

## Tasks / Subtasks

- [ ] Task 1: Write failing CI workflow test (conceptual) and Dockerfile (AC: 1, 2)
  - [ ] Create `services/temporal/Dockerfile`:
    ```dockerfile
    # Build stage
    FROM node:lts-alpine AS build
    WORKDIR /app
    COPY package*.json ./
    COPY shared/ ./shared/
    COPY services/temporal/ ./services/temporal/
    RUN npm ci
    RUN npm run build --workspace=@terminus-platform/temporal-worker

    # Runtime stage
    FROM node:lts-alpine AS runtime
    WORKDIR /app
    COPY --from=build /app/services/temporal/dist ./dist
    COPY --from=build /app/node_modules ./node_modules
    RUN addgroup -S appgroup && adduser -S appuser -G appgroup
    USER appuser
    ENTRYPOINT ["node", "dist/worker.js"]
    ```
  - [ ] Local Docker build test: `docker build -f services/temporal/Dockerfile -t terminus-platform-worker:test .`
  - [ ] Confirm image runs: `docker run --rm terminus-platform-worker:test` (expect connection error to Temporal — expected)
  - [ ] Commit: `feat(ci): add multi-stage Dockerfile for temporal-worker`

- [ ] Task 2: Implement `publish-images.yml` workflow (AC: 1)
  - [ ] Implement full workflow:
    ```yaml
    name: Publish Images
    on:
      push:
        branches: [main]
        paths:
          - "services/temporal/**"
    jobs:
      build-and-push:
        runs-on: ubuntu-latest
        permissions:
          contents: read
          packages: write
        steps:
          - uses: actions/checkout@v4
          - name: Log in to GHCR
            uses: docker/login-action@v3
            with:
              registry: ghcr.io
              username: ${{ github.actor }}
              password: ${{ secrets.GITHUB_TOKEN }}
          - name: Build and push
            uses: docker/build-push-action@v5
            with:
              context: .
              file: services/temporal/Dockerfile
              push: true
              tags: |
                ghcr.io/electricm0nk/terminus-platform-worker:${{ github.sha }}
                ghcr.io/electricm0nk/terminus-platform-worker:latest
    ```
  - [ ] YAML lint: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish-images.yml'))"`
  - [ ] Commit: `feat(ci): implement publish-images workflow for temporal-worker`

- [ ] Task 3: Trigger first CI build (AC: 3)
  - [ ] Push to `main` — confirm CI workflow triggers
  - [ ] Monitor workflow run: GitHub Actions UI
  - [ ] Confirm image pushed to GHCR: `ghcr.io/electricm0nk/terminus-platform-worker:{sha}`
  - [ ] Note the image digest from workflow output

- [ ] Task 4: Pin worker image digest (AC: 4, 5)
  - [ ] Get digest from Step 3 output
  - [ ] Update `helm/temporal-worker/values.yaml`:
    - Change `image.tag: latest` → `image.tag: {sha-from-ci}`
    - Change `image.pullPolicy: Always` → `image.pullPolicy: IfNotPresent`
  - [ ] Commit: `chore(helm): pin temporal-worker image to first CI build digest`
  - [ ] Trigger ArgoCD sync — confirm worker pod updates to pinned image
  - [ ] Document pinning step in completion notes (one-time N7 step)

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### Architecture Reference
Source: Architecture Step 6 (`.github/workflows/publish-images.yml`), N7 (image bootstrap gap)

### GHCR Authentication
`GITHUB_TOKEN` is automatically available in GitHub Actions workflows. No additional secrets configuration needed. Permissions block must include `packages: write`.

### Dockerfile Multi-Stage Strategy
- **Build stage**: installs all deps, compiles TypeScript → `dist/`
- **Runtime stage**: copies only `dist/` and production `node_modules` — no dev deps, no TypeScript compiler
- Result: minimal production image (~50-100MB vs ~500MB+)

### Path Filter
Workflow trigger `paths: ["services/temporal/**"]` — CI only runs on temporal worker changes. Other workspace changes do not trigger an image build.

---

## Architecture Alignment Note (Updated: 2026-04-06)

> **⚠️ DRIFT: This story was implemented using GitHub Actions, which violates the updated architecture rule.**
>
> **Rule (architecture.md — CI/CD Toolchain Decision Guide):**
> "Do NOT use GitHub Actions or any external CI service. GHCR pushes originate from within the network via Semaphore."
>
> **What was built:** `.github/workflows/publish-images.yml` using GitHub Actions runners + `GITHUB_TOKEN` for GHCR push.
>
> **What is required:** The Docker build and GHCR push must run via a Semaphore task on `trantor.internal`. A `scripts/publish.sh` should replace the GitHub Actions workflow, with a Semaphore Task Template configured to run it on push to `main`.
>
> **Remediation:** Create a follow-up story to migrate the GHCR push from GitHub Actions to a SemaphoreUI build task and remove `.github/workflows/publish-images.yml`.

### N7 Bootstrap Sequence (one-time)
After first successful CI build:
1. Get the SHA from the CI run or GHCR
2. Update `values.yaml`: `image.tag: <sha>`, `image.pullPolicy: IfNotPresent`
3. Commit and push → ArgoCD syncs with pinned image
4. This replaces the `latest` bootstrap tag from Story 3.1

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- CI workflows for `shared/types` or `shared/lib` (no images for these packages)
- ArgoCD auto-updater or image automation controller (out of scope for homelab)

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — only `GITHUB_TOKEN` (built-in), no external secrets |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — non-root runtime user, no secrets in image |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
