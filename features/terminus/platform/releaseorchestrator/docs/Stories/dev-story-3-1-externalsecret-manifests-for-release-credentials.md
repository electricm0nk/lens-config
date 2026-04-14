# Story 3.1: ExternalSecret Manifests for Release Credentials

Status: ready-for-dev

## Story

As a platform developer,
I want two `ExternalSecret` manifests in `k8s/` so that the Temporal worker can receive Semaphore and ArgoCD API tokens from Vault via ESO,
so that release credentials are never baked into the container image or Helm values and are rotated independently of deployments.

## Acceptance Criteria

1. `k8s/external-secret-release-semaphore.yaml` is created — syncs `release-semaphore-token` secret in `temporal` namespace from Vault
2. `k8s/external-secret-release-argocd.yaml` is created — syncs `release-argocd-token` secret in `temporal` namespace from Vault
3. Both manifests reference `ClusterSecretStore` named `vault-backend` (confirmed from existing manifests at `fourdogs-central/k8s/`)
4. Semaphore secret syncs: `SEMAPHORE_API_TOKEN` from `secret/terminus/default/semaphore/api-token`, `SEMAPHORE_PROJECT_ID` from `secret/terminus/default/semaphore/project-id`
5. ArgoCD secret syncs: `ARGOCD_API_TOKEN` from `secret/terminus/default/argocd/api-token`, `ARGOCD_BASE_URL` from `secret/terminus/default/argocd/base-url`
6. Existing `helm/temporal-worker/values.yaml` is updated to reference the new secrets as environment variable sources (`envFrom` or `env[].valueFrom.secretKeyRef`)
7. `kubectl apply --dry-run=client -f k8s/external-secret-release-semaphore.yaml` passes (if ESO CRDs are installed)
8. Vault paths listed above are NOT seeded by this story — that is operator responsibility (out of scope)

**Pre-condition:** Epic 1 complete. ESO is installed in the cluster with the `vault-backend` ClusterSecretStore configured (this is a cluster prereq, not created here). Story 3.1 is independent of Epic 2 and can be worked in parallel.

## Tasks / Subtasks

- [ ] Create `k8s/external-secret-release-semaphore.yaml` (AC: 1, 3, 4)
  - [ ] `namespace: temporal`
  - [ ] `secretStoreRef.name: vault-backend`, `kind: ClusterSecretStore`
  - [ ] Two keys: `SEMAPHORE_API_TOKEN` and `SEMAPHORE_PROJECT_ID`
  - [ ] Vault paths: `secret/terminus/default/semaphore/api-token` and `secret/terminus/default/semaphore/project-id`
- [ ] Create `k8s/external-secret-release-argocd.yaml` (AC: 2, 3, 5)
  - [ ] `namespace: temporal`
  - [ ] `secretStoreRef.name: vault-backend`, `kind: ClusterSecretStore`
  - [ ] Two keys: `ARGOCD_API_TOKEN` and `ARGOCD_BASE_URL`
  - [ ] Vault paths: `secret/terminus/default/argocd/api-token` and `secret/terminus/default/argocd/base-url`
- [ ] Update `helm/temporal-worker/values.yaml` to inject the new secrets as env vars (AC: 6)
  - [ ] Add `env` entries for `SEMAPHORE_API_TOKEN`, `SEMAPHORE_PROJECT_ID`, `ARGOCD_API_TOKEN`, `ARGOCD_BASE_URL` using `secretKeyRef`
  - [ ] Each entry references the correct secret name and key
- [ ] Run `kubectl apply --dry-run=client -f k8s/` (AC: 7) — if ESO CRDs are installed

## Dev Notes

### ExternalSecret — Semaphore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: release-semaphore-token
  namespace: temporal
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: release-semaphore-token
    creationPolicy: Owner
  data:
    - secretKey: SEMAPHORE_API_TOKEN
      remoteRef:
        key: secret/terminus/default/semaphore/api-token
        property: value
    - secretKey: SEMAPHORE_PROJECT_ID
      remoteRef:
        key: secret/terminus/default/semaphore/project-id
        property: value
```

### ExternalSecret — ArgoCD

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: release-argocd-token
  namespace: temporal
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: release-argocd-token
    creationPolicy: Owner
  data:
    - secretKey: ARGOCD_API_TOKEN
      remoteRef:
        key: secret/terminus/default/argocd/api-token
        property: value
    - secretKey: ARGOCD_BASE_URL
      remoteRef:
        key: secret/terminus/default/argocd/base-url
        property: value
```

### Vault KV v2 vs v1 Path Format

Check the existing `fourdogs-central/k8s/external-secret-app.yaml` for the `remoteRef.key` format used against this cluster's Vault. If Vault KV v2 is used without the `data/` prefix in ESO (depends on ESO Vault backend config), the path is `secret/terminus/default/semaphore/api-token` as shown above. If KV v2 prefix is required, paths would be `secret/data/terminus/default/semaphore/api-token`. Verify against the existing manifest pattern before committing.

### Helm values.yaml — Env Injection

The Temporal worker Helm chart uses `env` entries or `envFrom`. Check the existing `helm/temporal-worker/values.yaml` for the current pattern. If env vars are already injected from a different secret, follow the existing key format. Add:

```yaml
env:
  - name: SEMAPHORE_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: release-semaphore-token
        key: SEMAPHORE_API_TOKEN
  - name: SEMAPHORE_PROJECT_ID
    valueFrom:
      secretKeyRef:
        name: release-semaphore-token
        key: SEMAPHORE_PROJECT_ID
  - name: ARGOCD_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: release-argocd-token
        key: ARGOCD_API_TOKEN
  - name: ARGOCD_BASE_URL
    valueFrom:
      secretKeyRef:
        name: release-argocd-token
        key: ARGOCD_BASE_URL
```

### IR-4 Resolution

This story resolves Implementation Readiness Note IR-4: "Look up `ClusterSecretStore` name from `fourdogs-central/k8s/` for Story 3.1." The ClusterSecretStore name is **`vault-backend`**, confirmed from `TargetProjects/fourdogs/central/fourdogs-central/k8s/external-secret-app.yaml`.

### Operator Prerequisite (Out of Scope)

Before the worker can use these secrets, an operator must:
1. Seed `secret/terminus/default/semaphore/api-token` in Vault with `value: <token>`
2. Seed `secret/terminus/default/semaphore/project-id` in Vault with `value: <project-id>`
3. Seed `secret/terminus/default/argocd/api-token` in Vault with `value: <token>`
4. Seed `secret/terminus/default/argocd/base-url` in Vault with `value: https://<argocd-host>`

This work is documented in the architecture but does NOT belong in code for this story.

### References

- [Architecture: Credential Infrastructure](docs/terminus/platform/releaseorchestrator/architecture.md#credential-infrastructure)
- [Epics: Story 3.1](docs/terminus/platform/releaseorchestrator/epics.md#story-31-externalsecret-manifests-for-release-credentials)
- [Implementation Readiness: IR-4](docs/terminus/platform/releaseorchestrator/implementation-readiness.md)
- Existing ESO manifest: `TargetProjects/fourdogs/central/fourdogs-central/k8s/external-secret-app.yaml`

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List
