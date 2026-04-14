# Story 2.3: Register Semaphore ArgoCD Application

Status: ready-for-dev

## Story

As a platform operator,
I want an ArgoCD Application resource that tracks the Semaphore manifests from `terminus.infra` `main` branch,
So that all future changes to Semaphore manifests in git are automatically applied to the cluster via GitOps.

## Acceptance Criteria

1. **Given** the Semaphore manifests are committed at `platforms/k3s/manifests/terminus-infra/semaphoreui/` in `terminus.infra` `main`
   **When** the `semaphoreui.yaml` ArgoCD Application is applied (via App-of-Apps sync or kubectl)
   **Then** `kubectl get application semaphoreui -n argocd` shows `SYNC STATUS: Synced, HEALTH STATUS: Healthy`
2. All Semaphore resources (Deployment, Service, PVC, ExternalSecret, Certificate, Ingress) appear in ArgoCD UI as managed under the `semaphoreui` Application
3. A test commit to `platforms/k3s/manifests/terminus-infra/semaphoreui/` in `terminus.infra main` is auto-applied within the ArgoCD sync interval (typically 3 minutes)
4. **Smoke test:** `curl -sk https://semaphore.trantor.internal` returns login page (HTTP 200 or redirect to login)
5. **Smoke test:** Admin login to Semaphore succeeds using credentials from Vault

## Tasks

- [ ] Task 1: Create ArgoCD Application manifest
  - [ ] File: `platforms/k3s/argocd/apps/semaphoreui.yaml` in `terminus.infra`
  - [ ] `metadata.name: semaphoreui`, `namespace: argocd`
  - [ ] `metadata.labels: {app.kubernetes.io/part-of: terminus-infra}`
  - [ ] `spec.project: default`
  - [ ] `spec.source.repoURL`: `<terminus.infra repo URL>` (check existing apps in `platforms/k3s/argocd/apps/` for the pattern)
  - [ ] `spec.source.targetRevision: main`
  - [ ] `spec.source.path: platforms/k3s/manifests/terminus-infra/semaphoreui`
  - [ ] `spec.destination.server: https://kubernetes.default.svc`
  - [ ] `spec.destination.namespace: terminus-infra`
  - [ ] `spec.syncPolicy.automated: {selfHeal: true, prune: true}`
  - [ ] `spec.syncPolicy.syncOptions: [CreateNamespace=false]` (namespace already exists)
- [ ] Task 2: Trigger App-of-Apps to pick up the new Application
  - [ ] Commit `platforms/k3s/argocd/apps/semaphoreui.yaml` to `terminus.infra main`
  - [ ] If App-of-Apps does not auto-sync new files, force sync: `kubectl patch application root-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'`
  - [ ] Confirm `semaphoreui` Application appears in ArgoCD UI
- [ ] Task 3: Initial sync and verification
  - [ ] ArgoCD auto-syncs Semaphore resources from `terminus.infra main`
  - [ ] `kubectl get application semaphoreui -n argocd` — confirm `Synced, Healthy`
  - [ ] All 6 Semaphore resources visible in ArgoCD UI (Deployment, Service, PVC, ExternalSecret, Certificate, Ingress)
- [ ] Task 4: Smoke test — full end-to-end validation
  - [ ] `curl -sk https://semaphore.trantor.internal` — confirm login page returned (not 5xx)
  - [ ] Log in to `https://semaphore.trantor.internal` with `SEMAPHORE_ADMIN` + `SEMAPHORE_ADMIN_PASSWORD` from Vault
  - [ ] Confirm admin dashboard loads after login (no DB or auth errors)
  - [ ] Create a test Semaphore project/task and confirm it saves — validates DB connectivity in the container

## Dev Notes

### ArgoCD Application Source Pattern

Look at existing Application files in `platforms/k3s/argocd/apps/` (e.g., `keycloak.yaml`, `longhorn.yaml`, or similar) for the correct `repoURL` format used in this cluster. The format should be an HTTPS git URL to the `terminus.infra` repo.

### App-of-Apps Sync Behavior

The App-of-Apps watches the `platforms/k3s/argocd/apps/` directory. New files added there should be auto-picked up on the next sync interval. If auto-sync is not running, use the force-refresh patch in Task 2.

If there is no App-of-Apps (apps applied individually), apply directly:
```bash
kubectl apply -f platforms/k3s/argocd/apps/semaphoreui.yaml
```

### Sync Policy Decision

`prune: true` and `selfHeal: true` are intentional per TD-016:
- `prune`: Resources removed from git are deleted from cluster (clean state)
- `selfHeal`: Manual cluster changes that drift from git are reverted on next sync

This means: **never manually edit Semaphore resources in the cluster after this story**. All changes must go through `terminus.infra` manifests.

### Initial Sync vs. Manual Bootstrap

If Semaphore is already running (from manual `kubectl apply` testing in Stories 2.1/2.2), ArgoCD will adopt the existing resources. No need to delete and re-create unless the resources are in a broken state.

### Smoke Test Checklist

For the admin login smoke test, admin credentials come from Vault `secret/terminus/infra/semaphoreui`:
- Username: `SEMAPHORE_ADMIN` value
- Password: `SEMAPHORE_ADMIN_PASSWORD` value

If login fails: check `kubectl logs -n terminus-infra deployment/semaphoreui` for auth errors.

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — ArgoCD Application Manifest section]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-014 (ArgoCD manifest path), TD-016 (sync policy), TD-015 (App-of-Apps)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `platforms/k3s/argocd/apps/semaphoreui.yaml` (new, in `terminus.infra`)
