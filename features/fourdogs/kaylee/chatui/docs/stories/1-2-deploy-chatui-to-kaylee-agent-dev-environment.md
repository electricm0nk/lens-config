# Story 1.2: Deploy chatui to kaylee-agent dev environment

Status: ready-for-dev

## Story

As a developer,
I want the chatui served and reachable in the deployed dev environment,
so that I can use it to test Kaylee AI behaviour against live data without local port-forwarding.

## Acceptance Criteria

1. `COPY ui/ /app/ui/` is present in the kaylee-agent Dockerfile — the `ui/` directory is included in the container image
2. The Helm chart `values.yaml` has `devUI.enabled: false` (production default — off)
3. The Helm chart `deployment.yaml` conditionally sets `KAYLEE_DEV_UI_ENABLED=true` when `devUI.enabled: true`
4. `terminus.infra/platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` sets `devUI.enabled: true`
5. After ArgoCD syncs the dev deployment, `GET https://fourdogs-kaylee.trantor.internal/ui` returns 200 and renders the chat page
6. The production values file (`values.yaml` or `values-prod.yaml`) does NOT set `devUI.enabled: true`

## Tasks / Subtasks

- [ ] Add `COPY ui/ /app/ui/` to `Dockerfile` — must be after `WORKDIR /app` and before the final CMD (AC: 1)
  - [ ] Verify the directory is copied as `/app/ui/` which matches the `StaticFiles(directory="ui")` mount path in `main.py`
- [ ] Update `deploy/helm/fourdogs-kaylee-agent/values.yaml` — add dev UI section (AC: 2, 3)
  ```yaml
  devUI:
    enabled: false   # set to true in dev values only
  ```
- [ ] Update `deploy/helm/fourdogs-kaylee-agent/templates/deployment.yaml` — add conditional env var (AC: 3)
  ```yaml
  {{- if .Values.devUI.enabled }}
  - name: KAYLEE_DEV_UI_ENABLED
    value: "true"
  {{- end }}
  ```
  Add this block inside the container's `env:` section
- [ ] Update `terminus.infra/platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` — enable dev UI (AC: 4)
  ```yaml
  devUI:
    enabled: true
  ```
- [ ] Push kaylee-agent Dockerfile + chart changes → CI builds and pushes new image → ArgoCD syncs (AC: 5)
- [ ] Verify: `curl -I https://fourdogs-kaylee.trantor.internal/ui` returns `HTTP/2 200` (AC: 5)
- [ ] Verify: production `values.yaml` has `devUI.enabled: false` and no override exists in prod values (AC: 6)

## Dev Notes

- **Dockerfile COPY order:** Static assets don't need to precede deps install — place `COPY ui/ /app/ui/` just before the final `CMD`/`ENTRYPOINT` to avoid invalidating the pip cache layer
- **`StaticFiles` directory path:** `main.py` uses `StaticFiles(directory="ui", ...)` which resolves relative to the working directory — FastAPI app runs from `/app`, so `/app/ui/` is correct
- **Helm chart location:** `deploy/helm/fourdogs-kaylee-agent/` in the kaylee-agent repo (not `terminus.infra`)
- **terminus.infra values-dev.yaml path:** `terminus.infra/platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` — this is the correct per-env override file per Story 3.6 notes (GATEWAY_URL fix was committed here)
- **Two-repo change:** This story touches both `fourdogs-kaylee-agent` (Dockerfile + Helm chart) AND `terminus.infra` (values-dev.yaml). Coordinate PRs — merge kaylee-agent first (triggers CI to build new image), then merge terminus.infra change to apply values.
- **ArgoCD auto-sync:** Both `fourdogs-kaylee-agent` and `fourdogs-kaylee-infra` ArgoCD apps are on auto-sync. After CI pushes the image tag, ArgoCD picks up on the next sync. Values changes in `terminus.infra` also auto-sync.
- **No Ingress change needed:** `/ui` is served on the same FastAPI port 8000 as the API, behind the existing Traefik IngressRoute. No new route or cert required.
- **Production safety check:** Standard values.yaml default is `false`. No prod values file should set this. Add a comment in values.yaml: `# NEVER set true in production — dev tool only`

### Project Structure Notes

**fourdogs-kaylee-agent repo:**
- `Dockerfile` — add `COPY ui/ /app/ui/`
- `deploy/helm/fourdogs-kaylee-agent/values.yaml` — add `devUI.enabled: false`
- `deploy/helm/fourdogs-kaylee-agent/templates/deployment.yaml` — add conditional env var block

**terminus.infra repo:**
- `platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` — add `devUI.enabled: true`

### References

- Architecture (kaylee-agent): `docs/fourdogs/kaylee/architecture.md` — Deployment Pipeline, Helm values sketch
- Story 3.6 notes: `docs/fourdogs/kaylee/Stories/3-6-dev-release-pipeline-execution-and-verification.md` — confirms `terminus.infra/platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` path
- chatui architecture: `docs/fourdogs/kaylee/chatui/architecture.md` — D2 (StaticFiles), D6 (env var gate), Deployment section
- Tech Decisions: `docs/fourdogs/kaylee/chatui/tech-decisions.md` — TDL-006

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

**fourdogs-kaylee-agent repo:**
- `Dockerfile` (modified)
- `deploy/helm/fourdogs-kaylee-agent/values.yaml` (modified)
- `deploy/helm/fourdogs-kaylee-agent/templates/deployment.yaml` (modified)

**terminus.infra repo:**
- `platforms/k3s/helm/fourdogs-kaylee-agent/values-dev.yaml` (modified)
