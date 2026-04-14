# Story 4.1: Worker Deployment Validation and End-to-End Smoke

Status: backlog

## Story

As a platform operator,
I want the Go Temporal worker deployed to the `temporal` namespace and verified running with a live `ReleaseWorkflow` execution so that I can confirm the entire implementation is functional in the cluster,
so that the `terminus-platform-releaseorchestrator` initiative is fully complete and the deployment pipeline is production-ready.

## Acceptance Criteria

1. `kubectl get pods -n temporal` shows the `temporal-worker` pod in `Running` state with `READY 1/1` after all Epic 1–3 code is merged and CI deploys the new image
2. `kubectl get externalsecrets -n temporal` shows `release-semaphore-token` and `release-argocd-token` both in `Ready` state (ESO synced from Vault)
3. Worker pod logs show no error-level entries on startup (verify via `kubectl logs -n temporal <pod>`)
4. A live `ReleaseWorkflow` execution can be started via Temporal UI or CLI and reaches `WaitForArgoCDSync` state without error (Semaphore seeds must be in Vault before testing)
5. Temporal UI shows the workflow in `Running` or `Completed` state (not `Failed`)
6. All four env vars are present in the running pod: `SEMAPHORE_API_TOKEN`, `SEMAPHORE_PROJECT_ID`, `ARGOCD_API_TOKEN`, `ARGOCD_BASE_URL`

**Pre-condition:** All 11 stories (Epics 1–3) are merged to main in `TargetProjects/terminus/platform`. CI has built and pushed the new Go image. ArgoCD has synced the deployment. Operator has seeded Vault paths:
- `secret/terminus/default/semaphore/api-token`
- `secret/terminus/default/semaphore/project-id`
- `secret/terminus/default/argocd/api-token`
- `secret/terminus/default/argocd/base-url`

## Tasks / Subtasks

- [ ] Verify all Epic 1–3 stories are merged to main (pre-condition gate)
- [ ] Verify CI built new Go image successfully — check GHCR for a recent `terminus-platform-worker` tag
- [ ] Wait for ArgoCD to sync the `temporal-worker` application (or manually trigger sync)
- [ ] Verify pod is Running (AC: 1)
  ```bash
  kubectl get pods -n temporal -l app.kubernetes.io/name=temporal-worker
  ```
- [ ] Verify ExternalSecrets are Ready (AC: 2)
  ```bash
  kubectl get externalsecrets -n temporal
  ```
  - If ESO shows `SecretSyncedError`: check Vault paths are seeded and `vault-backend` ClusterSecretStore is functional
- [ ] Check startup logs for errors (AC: 3)
  ```bash
  kubectl logs -n temporal -l app.kubernetes.io/name=temporal-worker --tail=50
  ```
  - Expected: `"msg":"worker started"` or similar slog.Info output
  - Red flag: panic, `connection refused`, `fatal`
- [ ] Verify env vars are injected (AC: 6)
  ```bash
  kubectl exec -n temporal <pod-name> -- env | grep -E 'SEMAPHORE|ARGOCD'
  ```
- [ ] Trigger a `ReleaseWorkflow` execution (AC: 4, 5)
  - Via Temporal CLI:
    ```bash
    temporal workflow start \
      --task-queue terminus-platform \
      --type ReleaseWorkflow \
      --input '{"service_name":"fourdogs-central","provision_db":false}' \
      --workflow-id "release-test-$(date +%s)"
    ```
  - Or via Temporal UI: navigate to the `terminus-platform` task queue, start `ReleaseWorkflow` with the above input
- [ ] Observe workflow in Temporal UI — confirm it progresses past `SeedSecrets` without `Failed` status
- [ ] Record outcome in dev agent record below

## Dev Notes

### Input Shape

`ReleaseInput` JSON for the test execution:
```json
{
  "service_name": "fourdogs-central",
  "provision_db": false
}
```

Use `provision_db: false` for the first validation — this skips `ProvisionDatabase` and is less disruptive.

### ExternalSecret Not Ready

If `kubectl get externalsecrets -n temporal` shows the secrets are not Ready:
1. Check Vault is accessible: `kubectl exec -n terminus-infra <vault-agent-pod> -- vault status`
2. Check `vault-backend` ClusterSecretStore: `kubectl get clustersecretstores`
3. Seed the missing Vault paths as operator:
   ```bash
   vault kv put secret/terminus/default/semaphore/api-token value=<your-token>
   vault kv put secret/terminus/default/semaphore/project-id value=<your-project-id>
   vault kv put secret/terminus/default/argocd/api-token value=<your-token>
   vault kv put secret/terminus/default/argocd/base-url value=https://<your-argocd-host>
   ```
4. Force ESO refresh: `kubectl annotate externalsecret release-semaphore-token -n temporal force-sync=$(date +%s) --overwrite`

### Pod Not Starting

If the pod is failing:
1. Check rollout: `kubectl rollout status deployment -n temporal`
2. Check events: `kubectl events -n temporal --for pod/<pod-name>`
3. Common causes: wrong image tag in Helm values, image pull failure, env var missing

### Image Tag Check

The new Go image should be built by CI after Epic 1 is merged. Check `ghcr.io/electricm0nk/terminus-platform-worker` for a tag newer than the TypeScript build dates. If the tag is the old TypeScript image, CI hasn't run yet.

### Full Live Test (Optional — after all Vault seeds are in place)

To run a full end-to-end `ReleaseWorkflow` with all activities:
```json
{
  "service_name": "fourdogs-central",
  "provision_db": true
}
```

This will trigger all four activities in sequence against real Semaphore and ArgoCD. Only run this in a non-production context.

### This Story Does Not Write Code

This story is purely operational verification. No Go files, no Kubernetes manifests. If any AC fails because of an implementation bug discovered here, open a bug fix in the relevant Epic 2 story's implementation and fix it there.

### References

- [Architecture: Infrastructure & Deployment](docs/terminus/platform/releaseorchestrator/architecture.md#infrastructure--deployment)
- [Story 3.1: ExternalSecret Manifests](dev-story-3-1-externalsecret-manifests-for-release-credentials.md) — must be applied first
- [Story 2.2: ReleaseWorkflow Orchestration](dev-story-2-2-releaseworkflow-orchestration.md) — workflow under test

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
