# Story: transactions-2-04

## Context

**Feature:** transactions
**Sprint:** 2 — Deploy & Validate
**Priority:** high
**Estimate:** M

## User Story

As an operator, I want the CI/CD release infrastructure for `fourdogs-etailpet-trigger` registered
and reconciled — Semaphore templates, Ansible seed playbook, dev Vault path, and a successful first
Temporal release — so that every subsequent deployment follows the standard automated release pipeline.

## Acceptance Criteria

- [ ] Semaphore project 3 has all four required templates:
  - `seed-fourdogs-etailpet-trigger-secrets`
  - `db-provision-fourdogs-etailpet-trigger` (no-op stub — no database, but template required by release framework)
  - `verify-fourdogs-etailpet-trigger-dev-deploy`
  - `verify-fourdogs-etailpet-trigger-deploy`
- [ ] Templates are defined in `terminus.infra/tofu/environments/semaphore/main.tf` and applied via `tofu apply`
- [ ] `reconcile-semaphore.yml` pipeline run succeeds after template changes
- [ ] Ansible seed playbook `seed-fourdogs-etailpet-trigger-secrets.yml` exists and seeds both paths:
  - `secret/terminus/fourdogs/etailpet-api` (prod)
  - `secret/terminus/dev/fourdogs/etailpet-api` (dev)
- [ ] Dev Vault path `secret/terminus/dev/fourdogs/etailpet-api` exists with all required fields (same field names as prod, confirmed by transactions-0-01)
- [ ] First Temporal release completes successfully for `dev` environment: `SeedSecrets → ArgoCD → SmokeTest` all green (no `BootstrapDB`)
- [ ] Release input used: `{"ServiceName": "fourdogs-etailpet-trigger", "Environment": "dev", "ProvisionDB": false, "ImageTag": "<sha>"}`
- [ ] ArgoCD Application `fourdogs-etailpet-trigger` reports `Synced / Healthy` after release
- [ ] Temporal workflow ID `release-fourdogs-etailpet-trigger-dev-<sha8>` completes with status `Completed`

## Technical Notes

### Semaphore Template Naming (from cicd.md — MUST match exactly)

The Temporal `ReleaseWorkflow` looks up templates by exact name in Semaphore project 3. Wrong name
= silent no-op or hard failure. Casing matters. Names must be:

```
seed-fourdogs-etailpet-trigger-secrets
db-provision-fourdogs-etailpet-trigger
verify-fourdogs-etailpet-trigger-dev-deploy
verify-fourdogs-etailpet-trigger-deploy
```

Add these in `terminus.infra/tofu/environments/semaphore/main.tf`, following the pattern used by
`fourdogs-kaylee-agent` or `fourdogs-emailfetcher`. Run `tofu apply`, then `reconcile-semaphore.yml`.

### db-provision Stub

`etailpet-trigger` has no database — `ProvisionDB: false` in all releases. The
`db-provision-fourdogs-etailpet-trigger` template should be a minimal stub (e.g. echo + exit 0)
to satisfy any template existence check without doing anything. Confirm whether the release
framework requires the template to exist even when `ProvisionDB=false`.

### Ansible Seed Playbook

Create `seed-fourdogs-etailpet-trigger-secrets.yml` following the `seed-fourdogs-kaylee-agent-secrets.yml`
pattern. The playbook must write all required credential fields to both:
- `secret/terminus/fourdogs/etailpet-api` (prod path)
- `secret/terminus/dev/fourdogs/etailpet-api` (dev path)

Field names are determined by transactions-0-01. Run the playbook via Semaphore (project 3) before
the first release to satisfy the "Vault paths must exist before ESO syncs" rule.

### Dev Vault Path

The dev path `secret/terminus/dev/fourdogs/etailpet-api` must exist before the first release:
```bash
vault kv put secret/terminus/dev/fourdogs/etailpet-api \
  ETAILPET_API_KEY="<dev-key-or-test-value>" \
  ETAILPET_TRIGGER_URL="<url>"
# Adjust field names per transactions-0-01 spike output
```

If the seed playbook seeds both paths, this is satisfied automatically. Otherwise seed manually.

### First Temporal Release

Run via the standard release input (PascalCase field names — wrong casing is a silent no-op):

```json
{
  "ServiceName": "fourdogs-etailpet-trigger",
  "Environment": "dev",
  "ProvisionDB": false,
  "ImageTag": "<8-char-sha-from-ghcr>"
}
```

Verify with:
```bash
kubectl exec -n temporal deploy/temporal-server-admintools -- \
  /usr/local/bin/temporal workflow describe \
  --address 10.43.29.119:7233 \
  --namespace default \
  --workflow-id "release-fourdogs-etailpet-trigger-dev-<sha8>"
```

### Smoke Test Playbook

`verify-fourdogs-etailpet-trigger-dev-deploy` must verify the pod is healthy after ArgoCD sync.
Prefer HTTP check against the k8s liveness endpoint or `kubectl rollout status`. Do NOT rely on
`*.trantor.internal` hostnames for auth-sensitive checks (see cicd.md hostname policy).

A minimal smoke test:
```yaml
- name: Verify etailpet-trigger pod is Running
  shell: |
    kubectl rollout status deploy/fourdogs-etailpet-trigger -n fourdogs-central --timeout=120s
```

### Triage Reference

If release fails, match to these signatures from cicd.md:

| Failure | Likely cause | Fix |
|---|---|---|
| `no template with name "seed-fourdogs-etailpet-trigger-secrets"` | Template name mismatch or reconcile not run | Fix name in main.tf, run reconcile |
| `seed task failed` with Semaphore error | Missing Vault path/keys | Seed Vault paths first |
| `403` from Vault in seed | Insufficient token policy | Check `seed-reader` policy covers new path |
| `404` from Vault in seed | Path not created yet | Write both Vault paths manually then retry |
| ArgoCD `401` on sync | Expired ArgoCD JWT | Rotate `secret/terminus/default/argocd/api-token`, restart temporal-worker |

## Dependencies

**Blocked by:** transactions-2-01 (Docker image must be pushed to GHCR before first release),
transactions-2-02 (prod Vault path must exist before seed playbook can validate)
**Blocks:** transactions-2-03 (pod must be running before staging validation)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

- Does the Temporal `ReleaseWorkflow` require `db-provision-*` template to exist even when
  `ProvisionDB: false`? Confirm against emailfetcher (which also has no DB) or kaylee-agent release logs.
