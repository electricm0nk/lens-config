# Story 3.1: Worker Helm Chart + ArgoCD Application

Status: done

## Story

As a platform developer,
I want the Temporal worker deployed to k3s via a custom Helm chart managed by ArgoCD,
so that the TypeScript worker process runs as a Kubernetes Deployment with the correct ServiceAccount, resource limits, and ArgoCD sync wave ordering relative to the Temporal server.

## Acceptance Criteria

1. `helm/temporal-worker/Chart.yaml` created with appropriate name and version
2. `helm/temporal-worker/values.yaml` includes:
   - `image.repository: ghcr.io/electricm0nk/terminus-platform-worker`
   - `image.tag: latest` (bootstrap placeholder — Story 3.3 pins this to digest)
   - `image.pullPolicy: Always` (mutable tag during bootstrap)
   - `replicaCount: 1`
   - `resources.requests: {cpu: 100m, memory: 256Mi}`
   - `resources.limits: {cpu: 500m, memory: 512Mi}`
   - `temporal.address: temporal-frontend.temporal.svc.cluster.local:7233`
   - `temporal.namespace: default`
   - `temporal.taskQueue: terminus-platform`
   - Comment acknowledging `replicas: 1` rolling restart drops in-flight work — acceptable for homelab
3. `helm/temporal-worker/templates/deployment.yaml` references `temporal-worker` ServiceAccount (from Story 2.1), no credentials mounted directly
4. `helm/temporal-worker/templates/serviceaccount.yaml` references `serviceAccountName: temporal-worker`
5. ArgoCD `Application.yaml` at the resolved App-of-Apps location:
   - Points to `terminus.platform` repo, `helm/temporal-worker/` path
   - `spec.destination.namespace: temporal`
   - `argocd.argoproj.io/sync-wave: "3"`
   - `spec.syncPolicy.automated: {prune: true, selfHeal: true}`
6. Image bootstrap documented: first deploy requires manual image push to GHCR before Deployment reaches Running (bootstrap placeholder strategy: `terminus-platform-worker:bootstrap` using minimal node image)
7. Worker pod reaches `Running` state with bootstrap image (may log Temporal connection errors — expected until Story 3.2 produces real worker)

## Tasks / Subtasks

- [ ] Task 1: Create Helm chart scaffolding (AC: 1)
  - [ ] Create `helm/temporal-worker/Chart.yaml`:
    ```yaml
    apiVersion: v2
    name: temporal-worker
    description: Temporal worker deployment for terminus-platform
    version: 0.1.0
    appVersion: "0.1.0"
    ```
  - [ ] Commit: `chore(helm): scaffold temporal-worker Helm chart`

- [ ] Task 2: Create `values.yaml` (AC: 2)
  - [ ] Create `helm/temporal-worker/values.yaml` with all fields from ACs
  - [ ] Add comment on `replicas: 1`: rolling restart behaviour and homelab acceptance rationale
  - [ ] `image.pullPolicy: Always` — required while `image.tag: latest` is mutable
  - [ ] Commit: `feat(helm): add temporal-worker values.yaml`

- [ ] Task 3: Create Helm templates (AC: 3, 4)
  - [ ] Create `helm/temporal-worker/templates/deployment.yaml`:
    - Standard Deployment template referencing `values.yaml` fields
    - `serviceAccountName: temporal-worker`
    - No `envFrom`/`env` secrets mounted directly — future credentials come via ESO Secrets
    - `livenessProbe` stub pointing to `src/lib/health.ts` gRPC probe (Story 3.2 implements the script)
  - [ ] Create `helm/temporal-worker/templates/serviceaccount.yaml`:
    - Note: SA `temporal-worker` is created by Story 2.1 (`k8s/rbac-worker.yaml`) — this template creates a ServiceAccount reference only if a separate worker SA is needed, otherwise reference the existing one
  - [ ] Create `helm/temporal-worker/templates/_helpers.tpl` with chart name helpers
  - [ ] Commit: `feat(helm): add temporal-worker Deployment and ServiceAccount templates`

- [ ] Task 4: Create ArgoCD Application (AC: 5)
  - [ ] Create Application CR at the resolved App-of-Apps location (same location as Story 2.3)
  - [ ] Set `argocd.argoproj.io/sync-wave: "3"` (after Temporal server at wave 2)
  - [ ] Enable automated sync with prune and selfHeal
  - [ ] Source: `terminus.platform` repo, `helm/temporal-worker/` path
  - [ ] Commit ArgoCD Application to the infra repo

- [ ] Task 5: Image bootstrap and pod verification (AC: 6, 7)
  - [ ] Push a minimal bootstrap image: `docker pull node:lts-alpine && docker tag node:lts-alpine ghcr.io/electricm0nk/terminus-platform-worker:latest && docker push ghcr.io/electricm0nk/terminus-platform-worker:latest`
  - [ ] Trigger ArgoCD sync for worker Application
  - [ ] Confirm pod reaches `Running`: `kubectl get pods -n temporal -l app=temporal-worker`
  - [ ] Document bootstrap sequence in story completion notes for Story 3.3 hand-off

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/    (Helm chart)
TargetProjects/terminus/infra/terminus.infra/           (ArgoCD Application)
```

### ServiceAccount Reference
The `temporal-worker` ServiceAccount is created in Story 2.1 (`k8s/rbac-worker.yaml`). The worker Deployment references it via `serviceAccountName: temporal-worker`. Do NOT re-create the ServiceAccount in the Helm templates if it already exists as a raw manifest — reference it only.

### Sync Wave
Wave `"3"` — worker must not start before Temporal server is healthy (wave 2). String-quoted per Kubernetes annotation conventions.

### Image Bootstrap
Story 3.3 will replace the `latest` tag with a pinned digest from the CI build. The bootstrap strategy (minimal node image) is temporary. Document the one-time step to pin the digest after Story 3.3's first successful CI build.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- TypeScript worker implementation (Story 3.2)
- CI image build pipeline (Story 3.3)
- Any Temporal server configuration changes

---

## Architecture Alignment Note (Updated: 2026-04-06)

> **⚠️ DRIFT: This story is missing the required GHCR pull secret pattern. The worker Deployment will fail with `ImagePullBackOff` on any fresh cluster sync.**
>
> **What is missing (architecture.md — Consumer Service Deployment Pattern, Principle #6):**
> 1. `helm/temporal-worker/templates/external-secret-ghcr.yaml` at sync-wave `-1` syncing `ghcr-pull-secret` from `secret/terminus/default/ghcr/pull-token`
> 2. `imagePullSecrets: [{name: ghcr-pull-secret}]` in `helm/temporal-worker/templates/deployment.yaml`
>
> **What was built:** Helm chart Deployment has no `imagePullSecrets`. No GHCR ExternalSecret template exists in the chart.
>
> **Impact:** The worker pod will fail to pull `ghcr.io/electricm0nk/terminus-platform-worker` on any cluster where the pull secret does not already exist as a manually created k8s Secret.
>
> **Remediation:** Add `external-secret-ghcr.yaml` (wave -1) and `imagePullSecrets` to the Helm chart in a follow-up commit. Re-sync the ArgoCD Application after applying.

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no secrets in Helm chart values |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — no credentials in Deployment spec; ESO pattern for future secrets |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
