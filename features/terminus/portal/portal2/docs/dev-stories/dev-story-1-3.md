# Story 1.3: Helm Chart Adaptation for Vite Build

Status: done

## Story

As a platform operator,
I want the existing Helm chart at `deploy/helm/` adapted for the React/Vite image,
so that ArgoCD can deploy the new portal image without chart restructuring.

## Acceptance Criteria

1. `helm template terminus-portal deploy/helm/` output includes a Deployment referencing `ghcr.io/electricm0nk/terminus-portal`
2. Ingress resource specifies host `portal.trantor.internal`
3. Namespace is `terminus-portal`
4. No Docker Compose files exist anywhere in the repository
5. `helm lint deploy/helm/` passes with no errors or warnings

## Tasks / Subtasks

- [ ] Task 1: Inspect existing Helm chart (AC: #1–#5)
  - [ ] Read `deploy/helm/Chart.yaml`, `values.yaml`, and all templates
  - [ ] Note current image tag, host, namespace settings
- [ ] Task 2: Update `deploy/helm/values.yaml` (AC: #1, #2, #3)
  - [ ] Set `image.repository: ghcr.io/electricm0nk/terminus-portal`
  - [ ] Set `image.tag: latest` (CI will override with SHA)
  - [ ] Verify `ingress.hosts[0].host: portal.trantor.internal`
  - [ ] Verify `namespace: terminus-portal` (or equivalent chart field)
- [ ] Task 3: Remove Docker Compose if present (AC: #4)
  - [ ] `find . -name "docker-compose*.yml" -o -name "docker-compose*.yaml" | xargs rm -f`
- [ ] Task 4: Run lint and template checks (AC: #5, #1)
  - [ ] `helm lint deploy/helm/` — 0 errors, 0 warnings
  - [ ] `helm template terminus-portal deploy/helm/ | grep -E "ghcr.io|portal.trantor.internal|terminus-portal"` — all three present

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Existing chart:** `deploy/helm/` — confirmed present in repo (inspected via GitHub). Chart already working for v1 static HTML deployment.
**ArgoCD:** Reconciles from `ghcr.io/electricm0nk/terminus-portal` in namespace `terminus-portal`.
**Image tag strategy:** `values.yaml` sets `latest`; CI pipeline (Story 1.4) tags with `${GITHUB_SHA}` and `latest` on main merge. ArgoCD image updater or manual tag bump drives rollouts.

### Project Structure Notes

Files to modify:
```
terminus-portal/
└── deploy/
    └── helm/
        ├── Chart.yaml      (inspect only — may not need changes)
        ├── values.yaml     (primary edit target)
        └── templates/      (inspect — fix if Ingress host/namespace hardcoded)
```

### References

- NFR3 (k3s deployment, Helm chart, ArgoCD): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Deployment"
- Story 1.4 (CI pipeline) sets the image tag strategy

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
