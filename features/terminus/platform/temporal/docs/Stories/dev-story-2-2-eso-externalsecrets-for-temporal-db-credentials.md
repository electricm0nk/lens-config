# Story 2.2: ESO ExternalSecrets for Temporal DB Credentials

Status: done

## Story

As a platform developer,
I want ESO ExternalSecret resources that pull Temporal DB credentials from Vault into Kubernetes Secrets in the `temporal` namespace,
so that the Temporal Helm chart can reference k8s Secrets for database authentication without embedding plaintext credentials.

## Acceptance Criteria

1. **Resolve B1 (blocking):** ClusterSecretStore name looked up from domain architecture or crossplane ESO configuration. Name documented in implementation notes before writing manifests.
2. `k8s/external-secret-db.yaml` created: `ExternalSecret` named `temporal-db-credentials` in `temporal` namespace
   - `secretStoreRef.kind: ClusterSecretStore`, `name: {resolved-name}`
   - `remoteRef.key: secret/terminus/default/temporal/db-password`
   - `target.name: temporal-db-credentials`
   - `refreshInterval: 1h`
3. `k8s/external-secret-visibility-db.yaml` created: `ExternalSecret` named `temporal-visibility-db-credentials`
   - Same store, `remoteRef.key: secret/terminus/default/temporal/visibility-db-password`
   - `target.name: temporal-visibility-db-credentials`
   - `refreshInterval: 1h`
4. Both ExternalSecrets have sync wave annotation: `argocd.argoproj.io/sync-wave: "1"`
5. Both files pass YAML linting
6. After applying to cluster, k8s Secrets are created in `temporal` namespace

## Tasks / Subtasks

- [ ] Task 1: Resolve B1 — ClusterSecretStore name (AC: 1)
  - [ ] Read `docs/terminus/infra/crossplane/architecture.md` for ESO ClusterSecretStore name
  - [ ] Alternatively: `kubectl get clustersecretstore -A` against the k3s cluster
  - [ ] Document the resolved name in a comment at top of each ExternalSecret manifest
  - [ ] Example: `# ClusterSecretStore: vault-backend (resolved from crossplane initiative)`

- [ ] Task 2: Create `k8s/external-secret-db.yaml` (AC: 2, 4, 5)
  - [ ] Create manifest with:
    - `apiVersion: external-secrets.io/v1beta1`
    - `kind: ExternalSecret`
    - `name: temporal-db-credentials`, `namespace: temporal`
    - `spec.refreshInterval: 1h`
    - `spec.secretStoreRef.kind: ClusterSecretStore`, `name: {resolved-name}`
    - `spec.target.name: temporal-db-credentials`
    - `spec.data[0].secretKey: password`
    - `spec.data[0].remoteRef.key: secret/terminus/default/temporal/db-password`
    - `spec.data[0].remoteRef.property: password`
    - `argocd.argoproj.io/sync-wave: "1"` annotation
  - [ ] YAML lint: `python3 -c "import yaml; yaml.safe_load(open('k8s/external-secret-db.yaml'))"`
  - [ ] Commit: `feat(k8s): add ESO ExternalSecret for temporal DB credentials`

- [ ] Task 3: Create `k8s/external-secret-visibility-db.yaml` (AC: 3, 4, 5)
  - [ ] Create manifest mirroring Task 2 but for `temporal-visibility-db-credentials`
  - [ ] `remoteRef.key: secret/terminus/default/temporal/visibility-db-password`
  - [ ] YAML lint
  - [ ] Commit: `feat(k8s): add ESO ExternalSecret for temporal visibility DB credentials`

- [ ] Task 4: Verify k8s Secrets are synced (AC: 6)
  - [ ] Apply manifests: `kubectl apply -f k8s/external-secret-db.yaml -f k8s/external-secret-visibility-db.yaml`
  - [ ] Verify Secrets exist: `kubectl get secret temporal-db-credentials -n temporal`
  - [ ] Verify Secrets exist: `kubectl get secret temporal-visibility-db-credentials -n temporal`

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### B1 Resolution
Adversarial finding B1: ClusterSecretStore name was unspecified in architecture. Agent must resolve from:
1. `docs/terminus/infra/crossplane/architecture.md` (crossplane initiative docs in this repo)
2. `kubectl get clustersecretstore -A` (live cluster query)

The resolved name MUST be documented in a comment at the top of each ExternalSecret file before writing the manifest body.

### ESO API Reference
- API group: `external-secrets.io/v1beta1`
- `Kind: ExternalSecret`
- `spec.secretStoreRef.kind: ClusterSecretStore` (cluster-scoped, not namespace-scoped)

### Sync Wave
Wave `"1"` — same wave as namespace/RBAC (Story 2.1). ESO syncs secrets early so they exist before Temporal server pods start.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- Vault secret creation (Story 1.2)
- Temporal Helm chart values referencing these Secrets (Story 2.3)

---

## Architecture Alignment Note (Updated: 2026-04-06)

> **⚠️ DRIFT: Vault paths use `default/temporal/` and sync-wave is `"1"` — both deviate from the updated architecture.**
>
> **Issue 1 — Vault path:**
> - What was built: `remoteRef.key: secret/terminus/default/temporal/db-password`
> - Architecture hierarchy requires: `secret/terminus/dev/temporal/db-password`
> - `secret/terminus/default/` is reserved for shared cross-service resources (GHCR pull token only)
>
> **Issue 2 — Sync wave:**
> - What was built: `argocd.argoproj.io/sync-wave: "1"`
> - Architecture mandates: wave `"0"` for service credential ExternalSecrets (wave `-1` is GHCR pull secret, wave `"0"` is app credentials, wave `"1+"` is Deployments)
> - The Temporal server Deployment is at wave `"2"` (Story 2.3), so wave `"1"` for ESO does work in practice, but it is inconsistent with the canonical pattern.
>
> **Remediation:**
> 1. Move Vault secrets to `secret/terminus/dev/temporal/` (Story 1.2 remediation)
> 2. Update `remoteRef.key` in both ExternalSecret manifests to `secret/terminus/dev/temporal/db-password` and `secret/terminus/dev/temporal/visibility-db-password`
> 3. Update sync-wave annotation from `"1"` to `"0"` on both ExternalSecrets

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no plaintext credentials; all via Vault/ESO |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| Security first | Org Art. 9 | ✅ PASS — secrets delivered via ESO, never in manifests |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
