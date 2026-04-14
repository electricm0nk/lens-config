# Temporal Deployment Runbook
**Initiative:** terminus-platform-temporal | **Track:** tech-change
**Environment:** terminus homelab k3s cluster
**Date:** 2026-03-31

---

## Overview

Deploys Temporal workflow engine to k3s as a platform-layer service. Depends on k3s, Patroni PostgreSQL, and Vault being operational.

**Repos involved:**
- `terminus.platform` — all source code and Helm charts (develop-first integration per Art. 5)
- `terminus.infra` — ArgoCD Application manifests (PR #62)

---

## Prerequisites

- [ ] `terminus-infra-k3s`: k3s cluster running
- [ ] `terminus-infra-postgres`: Patroni cluster running at 10.0.0.56:5432
- [ ] `terminus-infra-secrets`: Vault running at vault.trantor.internal, ESO ClusterSecretStore `vault-backend` healthy
- [ ] DNS: `temporal-ui.trantor.internal` → 10.0.0.126 (Traefik LB)
- [ ] terminus.infra PR #62 merged

### GitHub PATs Required

Three separate tokens are needed. Create them before starting.

| Token | Scope | Used For |
|-------|-------|----------|
| ArgoCD repo credential | Classic PAT — `repo` | ArgoCD reading terminus.platform (private repo) |
| GHCR push (bootstrap) | Classic PAT — `write:packages` | Pushing bootstrap worker image to GHCR |
| GHCR pull secret | Classic PAT — `read:packages` | k3s nodes pulling worker image from GHCR |

**ArgoCD repo secret** (run once on k3s control plane):
```bash
sudo k3s kubectl create secret generic argocd-repo-terminus-platform \
  -n argocd \
  --from-literal=type=git \
  --from-literal=url=https://github.com/electricm0nk/terminus.platform.git \
  --from-literal=username=x-access-token \
  --from-literal=password=<classic-pat-repo-scope>
sudo k3s kubectl label secret argocd-repo-terminus-platform -n argocd \
  argocd.argoproj.io/secret-type=repository
```

**GHCR pull secret** (run once on k3s control plane after temporal namespace exists):
```bash
sudo k3s kubectl create secret docker-registry ghcr-pull-secret \
  -n temporal \
  --docker-server=ghcr.io \
  --docker-username=electricm0nk \
  --docker-password=<classic-pat-read-packages-scope>
sudo k3s kubectl patch serviceaccount temporal-worker -n temporal \
  -p '{"imagePullSecrets": [{"name": "ghcr-pull-secret"}]}'
```

---

## Step 1 — Patroni DB Provisioning (Story 1.2) ⚠️ CLUSTER-INTERACTIVE

Must complete before ArgoCD sync activates.

```bash
# 1. Get Patroni admin password from Vault
PGPASSWORD=$(vault kv get -field=password secret/terminus/default/patroni/admin-password)

# 2. Generate temporal user password
TEMPORAL_PASSWORD=$(openssl rand -base64 32)

# 3. Seed Vault BEFORE running SQL
vault kv put secret/terminus/default/temporal/db-password password="$TEMPORAL_PASSWORD"
vault kv put secret/terminus/default/temporal/visibility-db-password password="$TEMPORAL_PASSWORD"

# 4. Run provisioning script
PGPASSWORD=$PGPASSWORD psql -h 10.0.0.56 -U admin \
  -v temporal_password="$TEMPORAL_PASSWORD" \
  -f TargetProjects/terminus/platform/terminus.platform/docs/scripts/db-provision-temporal.sql

# 5. Verify
PGPASSWORD=$PGPASSWORD psql -h 10.0.0.56 -U admin -c "\l" | grep temporal
```

---

## Step 2 — Bootstrap Worker Image ⚠️ CLUSTER-INTERACTIVE

Worker Deployment will ImagePullBackOff without an image at GHCR. Push a bootstrap image before ArgoCD syncs the worker:

```bash
# Pull a minimal Node.js image as bootstrap
docker pull node:lts-alpine
docker tag node:lts-alpine ghcr.io/electricm0nk/terminus-platform-worker:latest
docker push ghcr.io/electricm0nk/terminus-platform-worker:latest
```

After Story 3.3 CI runs successfully, update `helm/temporal-worker/values.yaml`:
```yaml
image.tag: sha256:<digest-from-first-ci-build>
image.pullPolicy: IfNotPresent
```

---

## Step 3 — Merge terminus.infra PR #62

Once Steps 1 and 2 are complete:
- Merge [terminus.infra#62](https://github.com/electricm0nk/terminus.infra/pull/62)
- ArgoCD root-app will pick up `temporal-infra.yaml`, `temporal-server.yaml`, `temporal-worker.yaml`

---

## Step 4 — Monitor Sync Progression

```bash
# Watch all Applications
kubectl get applications -n argocd -w | grep temporal

# Watch temporal namespace pods
kubectl get pods -n temporal -w
```

**Expected progression:**
1. `temporal-infra` syncs: namespace, RBAC, ESO ExternalSecrets, Certificate, Ingress
2. ESO syncs secrets from Vault (may take up to 1h per refreshInterval; trigger manual sync: `kubectl annotate externalsecret -n temporal temporal-db-credentials force-sync=$(date +%s)`)
3. `temporal-server` syncs: admintools Job runs schema migrations, server pods start
4. `temporal-worker` syncs: worker pod starts, connects to Temporal frontend

---

## Step 5 — Verify Temporal Server Health

```bash
# All server pods running
kubectl get pods -n temporal

# Schema migration completed
kubectl get jobs -n temporal | grep admintools

# Frontend service accessible
kubectl exec -n temporal deploy/temporal-admintools -- \
  tctl --namespace default namespace describe
```

---

## Step 6 — Run Smoke Test

See [smoke-test-results.md](smoke-test-results.md) for the full procedure.

Quick check:
```bash
# Trigger test workflow
kubectl exec -n temporal -it deploy/temporal-admintools -- \
  tctl workflow run \
    --namespace default \
    --task_queue terminus-platform \
    --workflow_type DailyBriefingWorkflow \
    --execution_timeout 60 \
    --input '{"schema":{"date":"2026-03-31","timezone":"America/New_York"},"context":{"date":"2026-03-31T00:00:00Z","timezone":"America/New_York","config":{}},"data":{}}'
```

---

## Troubleshooting

### ESO ExternalSecret SYNCED=False
```bash
kubectl describe externalsecret -n temporal temporal-db-credentials
# Usually: Vault secret not seeded (Step 1 not run) or vault-backend ClusterSecretStore unhealthy
kubectl get clustersecretstore vault-backend
```

### admintools Job fails (schema migration error)
```bash
kubectl logs -n temporal job/temporal-admintools
# Usually: DB not created (Step 1 not run), wrong credentials, or wrong DB name
```

### Worker pod CrashLoopBackOff
```bash
kubectl logs -n temporal -l app.kubernetes.io/name=temporal-worker
# Usually: image not pushed (Step 2 not done) or Temporal server not yet healthy
```

### Worker can't reach Temporal frontend
```bash
kubectl exec -n temporal -l app.kubernetes.io/name=temporal-worker -- \
  node /app/services/temporal/dist/lib/health.js
# Exit 0 = connected, Exit 1 = cannot reach temporal-frontend:7233
```

---

## Defects Logged (review before blast-and-repave)

| ID | Story | Severity | Description |
|----|-------|----------|-------------|
| D1 | 2.4 | Low | Story AC said `kind: IngressRoute` (Traefik CRD). Used standard `networking.k8s.io/v1 Ingress` to match cluster pattern (semaphoreui). Verify Traefik routes correctly. |
| D2 | 1.1 | Low | `workspace:*` protocol not supported by npm. Changed to `"*"` for workspace deps. |
| D3 | 2.3 | Low | Story bundled 2.3 helm values commit with 2.4, 3.2, 3.3 implementation (single commit in terminus.platform). Individual story provenance in git history may be unclear. |
| D4 | 4.1 | Medium | Smoke test (Story 4.1) requires live cluster interaction — cannot be automated in batch run. Procedure documented in smoke-test-results.md; must be executed manually post-deployment. |

---

## Version Reference

| Component | Version |
|-----------|---------|
| Temporal Helm chart | 1.0.0-rc.3 (appVersion 1.30.2) |
| TypeScript | ^5.4.0 |
| Node.js | 20 LTS |
| @temporalio/* | ^1.15.0 |
