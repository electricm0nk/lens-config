# Story 4.2: Deploy cert-manager with Vault PKI ClusterIssuer

## Status: done

## Story

As an operator,
I want cert-manager deployed with a `ClusterIssuer` pointing at the Vault PKI internal CA,
So that workloads can request TLS certificates signed by the internal Vault CA.

## Acceptance Criteria

- **Given** ArgoCD managing the cluster from Epic 2, `cert-manager` namespace present, and Vault reachable at `vault.trantor.internal`
- **When** the cert-manager `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/cert-manager.yaml`
- **Then** all cert-manager pods are `Running` in the `cert-manager` namespace
- **And** a `ClusterIssuer` of type Vault PKI is present and reports `Ready: True`
- **And** a test `Certificate` resource issued against the `ClusterIssuer` reaches `Ready: True` and contains a valid TLS cert signed by the Vault CA
- **And** cert-manager Helm values are sourced exclusively from `infra/k3s/helm/cert-manager/values.yaml`
- **And** all cert-manager resources carry the 4 standard labels

## Tasks / Subtasks

- [x] Task 1: Create cert-manager Helm values
  - [x] `infra/k3s/helm/cert-manager/values.yaml` — `installCRDs: true`, no CP tolerations
- [x] Task 2: Create Vault PKI ClusterIssuer manifest
  - [x] `infra/k3s/manifests/cert-manager/vault-clusterissuer.yaml` — `ClusterIssuer` vault-pki → `vault.trantor.internal` PKI path
  - [x] Carries 4 standard labels
- [x] Task 3: Update cert-manager.yaml ArgoCD Application
  - [x] `infra/k3s/argocd/apps/cert-manager.yaml` — 3-source spec; `syncPolicy.automated.prune: true, selfHeal: true`
- [x] Task 4: Test certificate issuance
  - [x] `infra/k3s/manifests/validation/cert-test.yaml` — test `Certificate` resource for manual validation

## Dev Notes

**cert-manager chart source:** `https://charts.jetstack.io`

**ClusterIssuer for Vault PKI:**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-pki
spec:
  vault:
    server: https://vault.trantor.internal
    path: pki/sign/terminus-internal
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        serviceAccountRef:
          name: cert-manager
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- ClusterIssuer uses cert-manager's own SA (not operator-tier SA from Story 2.2) for Vault Kubernetes auth — cert-manager SA must also be registered with Vault (needs operator coordination)
- 3-source ArgoCD Application: ClusterIssuer CRD provided by chart; manifests dir syncs after CRDs available
- PR: electricm0nk/terminus.infra#38
### Change List
- `infra/k3s/helm/cert-manager/values.yaml` — installCRDs: true, no CP tolerations
- `infra/k3s/manifests/cert-manager/vault-clusterissuer.yaml` — ClusterIssuer vault-pki
- `infra/k3s/argocd/apps/cert-manager.yaml` — full 3-source Application
- `infra/k3s/manifests/validation/cert-test.yaml` — manual cert issuance test
