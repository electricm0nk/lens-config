# Story 2.1: Create Semaphore Deployment, Service, and PVC Manifests

Status: ready-for-dev

## Story

As a platform operator,
I want the Semaphore Deployment, Service, and PersistentVolumeClaim manifests committed to `terminus.infra`,
So that ArgoCD can deliver the Semaphore pod to the cluster with persistent storage and correct credentials.

## Acceptance Criteria

1. **Given** the `semaphoreui-secrets` k8s Secret is synced (Story 1.2 complete) AND the `semaphore` Postgres DB exists (Story 1.1 complete)
   **When** the manifests are committed to `platforms/k3s/manifests/terminus-infra/semaphoreui/` and ArgoCD syncs them
   **Then** a `semaphoreui` pod is `Running` in the `terminus-infra` namespace
2. PVC `semaphoreui-data` is `Bound` and mounted at `/tmp/semaphore` in the pod
3. `kubectl logs -n terminus-infra deployment/semaphoreui` shows no fatal errors (DB connection succeeded)
4. `kubectl exec -n terminus-infra deployment/semaphoreui -- wget -q -O- http://localhost:3000/api/ping` returns `{"message":"PONG"}` (or HTTP 200)

## Tasks

- [ ] Task 1: Create `pvc.yaml`
  - [ ] File: `platforms/k3s/manifests/terminus-infra/semaphoreui/pvc.yaml`
  - [ ] `metadata.name: semaphoreui-data`, `namespace: terminus-infra`
  - [ ] `spec.storageClassName: local-path`, `accessModes: [ReadWriteOnce]`
  - [ ] `spec.resources.requests.storage: 1Gi`
  - [ ] Labels: `app.kubernetes.io/name: semaphoreui`, `app.kubernetes.io/part-of: terminus-infra`
- [ ] Task 2: Create `deployment.yaml`
  - [ ] File: `platforms/k3s/manifests/terminus-infra/semaphoreui/deployment.yaml`
  - [ ] `metadata.name: semaphoreui`, `namespace: terminus-infra`
  - [ ] `spec.replicas: 1`
  - [ ] Container: `image: semaphoreui/semaphore:latest`, `imagePullPolicy: Always`
  - [ ] `envFrom`: `secretRef.name: semaphoreui-secrets` — injects all `SEMAPHORE_*` credentials
  - [ ] Plain `env` vars (non-secret): `SEMAPHORE_DB_DIALECT=postgres`, `SEMAPHORE_DB_HOST=10.0.0.56`, `SEMAPHORE_DB_PORT=5432`, `SEMAPHORE_DB_NAME=semaphore`, `SEMAPHORE_DB_USER=semaphore`
  - [ ] VolumeMount: `/tmp/semaphore` from volume `semaphoreui-data`
  - [ ] Volume: `persistentVolumeClaim.claimName: semaphoreui-data`
  - [ ] Resources: `requests: {cpu: 100m, memory: 256Mi}`, `limits: {memory: 512Mi}` (no CPU limit)
  - [ ] ReadinessProbe: `httpGet.path: /api/ping`, `port: 3000`, `initialDelaySeconds: 15`, `periodSeconds: 10`
  - [ ] Labels: `app.kubernetes.io/name: semaphoreui`, `app.kubernetes.io/part-of: terminus-infra`
- [ ] Task 3: Create `service.yaml`
  - [ ] File: `platforms/k3s/manifests/terminus-infra/semaphoreui/service.yaml`
  - [ ] `metadata.name: semaphoreui`, `namespace: terminus-infra`, `spec.type: ClusterIP`
  - [ ] `spec.ports[]: {port: 3000, targetPort: 3000, protocol: TCP}`
  - [ ] `spec.selector: {app.kubernetes.io/name: semaphoreui}`
- [ ] Task 4: Verify deployment health (after ArgoCD sync — or manual `kubectl apply` for initial bootstrap)
  - [ ] `kubectl get pod -n terminus-infra -l app.kubernetes.io/name=semaphoreui` — confirm `Running`
  - [ ] `kubectl get pvc semaphoreui-data -n terminus-infra` — confirm `Bound`
  - [ ] `kubectl logs -n terminus-infra deployment/semaphoreui` — no fatal errors, no DB connection refusals
  - [ ] `kubectl exec -n terminus-infra deployment/semaphoreui -- wget -q -O- http://localhost:3000/api/ping` — confirm PONG

## Dev Notes

### Credential Injection Pattern

Use `envFrom: secretRef` to inject all fields from `semaphoreui-secrets` at once (`SEMAPHORE_ADMIN`, `SEMAPHORE_ADMIN_PASSWORD`, `SEMAPHORE_ADMIN_EMAIL`, `SEMAPHORE_DB_PASS`, `SEMAPHORE_ACCESS_KEY_ENCRYPTION`). Do NOT use `valueFrom: secretKeyRef` for individual fields alongside `envFrom` — this creates duplicate injection for the same secret and is redundant. The `SEMAPHORE_DB_PASS` comes from `envFrom` (Advisory note from adversarial review: use `envFrom` exclusively).

### Plain DB Environment Variables

These are NOT secrets and must be in plain `env` (not from the Secret):
- `SEMAPHORE_DB_DIALECT=postgres`
- `SEMAPHORE_DB_HOST=10.0.0.56` (Patroni direct IP — accepted homelab constraint)
- `SEMAPHORE_DB_PORT=5432`
- `SEMAPHORE_DB_NAME=semaphore`
- `SEMAPHORE_DB_USER=semaphore`

### Readiness Probe Endpoint

Semaphore exposes `/api/ping` → `{"message":"PONG"}`. If this endpoint differs in the installed version, check the Semaphore docs or test manually with `wget -O- http://localhost:3000/api/ping`. The readiness probe prevents traffic routing until Semaphore is actually ready (DB migration complete, etc.).

### PVC and local-path StorageClass

The `local-path` PVC binds to the scheduling node. In a single-node or small homelab cluster, this is fine. The pod and its data are co-located. If the node changes (e.g., after node failure/rebuild), the PVC is lost — acceptable for homelab. No pod affinity rules needed for the current cluster size.

### Deployment Sequencing

This story depends on both Story 1.1 and 1.2 being complete:
- Without `semaphoreui-secrets`, the pod will fail to start (missing envFrom secret)
- Without the Postgres DB, the pod will fail at DB migration with a connection error

If testing these manifests before the secrets/DB are ready, the pod will crash-loop. That is expected.

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — Deployment Spec section]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-001 (per-resource split), TD-007 (PVC size), TD-002 (manifest path)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `platforms/k3s/manifests/terminus-infra/semaphoreui/pvc.yaml` (new, in `terminus.infra`)
- `platforms/k3s/manifests/terminus-infra/semaphoreui/deployment.yaml` (new, in `terminus.infra`)
- `platforms/k3s/manifests/terminus-infra/semaphoreui/service.yaml` (new, in `terminus.infra`)
