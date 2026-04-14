# Story 4.3: Deploy ESO with Vault SecretStore

## Status: done

## Story

As an operator,
I want External Secrets Operator deployed with a `SecretStore` configured for Vault Kubernetes auth,
So that workloads can retrieve secrets from Vault KV v2 without storing credentials in source control.

## Acceptance Criteria

- **Given** the ESO bootstrap secret from Story 4.1 present in `external-secrets` namespace
- **When** the ESO `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/external-secrets.yaml`
- **Then** all ESO pods are `Running` in the `external-secrets` namespace and carry no `NoSchedule` control-plane toleration
- **And** a `ClusterSecretStore` (or `SecretStore`) is present in `external-secrets` and reports `Valid: True`
- **And** a test `ExternalSecret` successfully syncs a known Vault KV v2 secret into a Kubernetes `Secret`
- **And** ESO Helm values are sourced exclusively from `infra/k3s/helm/external-secrets/values.yaml`

## Tasks / Subtasks

- [x] Task 1: Create ESO Helm values
  - [x] `infra/k3s/helm/external-secrets/values.yaml` — no CP tolerations (controller + webhook + certController)
- [x] Task 2: Create Vault ClusterSecretStore manifest
  - [x] `infra/k3s/manifests/external-secrets/vault-secretstore.yaml` — `ClusterSecretStore` vault-backend, KV v2, Kubernetes auth via external-secrets-sa
  - [x] Carries 4 standard labels
- [x] Task 3: Update external-secrets.yaml ArgoCD Application
  - [x] `infra/k3s/argocd/apps/external-secrets.yaml` — 3-source spec; `syncPolicy.automated.prune: true, selfHeal: true`
- [x] Task 4: Test ExternalSecret sync
  - [x] `infra/k3s/manifests/validation/external-secret-test.yaml` — ExternalSecret syncing `secret/terminus/test` → k8s Secret for manual validation

## Dev Notes

**ESO chart source:** `https://charts.external-secrets.io`

**ClusterSecretStore for Vault KV v2:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: https://vault.trantor.internal
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

**No control-plane toleration (architecture constraint):** ESO pods must schedule on workers only. Confirm `values.yaml` does not set `tolerations` for `node-role.kubernetes.io/control-plane`.

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- Explicit `tolerations: []` on all ESO components (controller, webhook, certController) — architecture constraint prevents any CP scheduling
- `serviceAccountRef.name: external-secrets-sa` — uses operator-tier SA registered with Vault in Story 4.1
- PR: electricm0nk/terminus.infra#39; Epic 4 PR: #40
### Change List
- `infra/k3s/helm/external-secrets/values.yaml` — no CP tolerations across all components
- `infra/k3s/manifests/external-secrets/vault-secretstore.yaml` — ClusterSecretStore vault-backend
- `infra/k3s/argocd/apps/external-secrets.yaml` — full 3-source Application
- `infra/k3s/manifests/validation/external-secret-test.yaml` — ExternalSecret validation manifest
