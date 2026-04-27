# Story: sales-2-02

## Context

**Feature:** fourdogs-etailpet-api-sales
**Sprint:** 2 — Deploy and Validate
**Priority:** high
**Estimate:** S

## User Story

As an operator, I want an ESO `ExternalSecret` for the `fourdogs-etailpet-sales-trigger` namespace
that pulls the same Vault credentials as the transactions worker, so that the sales worker can
authenticate to eTailPet without duplicating Vault secrets.

## Acceptance Criteria

- [ ] ESO `ExternalSecret` named `fourdogs-etailpet-api-sales-secrets` in namespace `fourdogs-etailpet-sales-trigger`
- [ ] ExternalSecret maps Vault `CLIENT_ID` → k8s Secret key `ETAILPET_CLIENT_ID`
- [ ] ExternalSecret maps Vault `CLIENT_SECRET` → k8s Secret key `ETAILPET_CLIENT_SECRET`
- [ ] Vault path is `secret/terminus/fourdogs/etailpet-api` (same as transactions worker)
- [ ] ESO `ClusterSecretStore` or `SecretStore` reference matches the existing store used by transactions worker
- [ ] k8s Secret `fourdogs-etailpet-api-sales-secrets` is created and synced before the Deployment is scheduled
- [ ] ESO sync confirmed: `kubectl get externalsecret -n fourdogs-etailpet-sales-trigger` shows `Ready`

## Technical Notes

**ExternalSecret manifest:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: fourdogs-etailpet-api-sales-secrets
  namespace: fourdogs-etailpet-sales-trigger
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: <cluster-secret-store-name>   # match transactions worker
    kind: ClusterSecretStore
  target:
    name: fourdogs-etailpet-api-sales-secrets
    creationPolicy: Owner
  data:
    - secretKey: ETAILPET_CLIENT_ID
      remoteRef:
        key: secret/terminus/fourdogs/etailpet-api
        property: CLIENT_ID
    - secretKey: ETAILPET_CLIENT_SECRET
      remoteRef:
        key: secret/terminus/fourdogs/etailpet-api
        property: CLIENT_SECRET
```

Check the existing `ExternalSecret` for `fourdogs-etailpet-trigger` to confirm the correct
`ClusterSecretStore` name and `refreshInterval` convention before committing this manifest.

**Vault path reuse:** No new Vault secret required. The existing
`secret/terminus/fourdogs/etailpet-api` path contains `CLIENT_ID` and `CLIENT_SECRET` which
are valid for both `pos` and `www` auth (confirmed by sales-0-01 spike).

## Dependencies

**Blocked by:** sales-0-01 (confirm Vault creds work for www auth)
**Blocks:** sales-2-03

## Definition of Done

- [ ] ExternalSecret manifest committed to GitOps repo
- [ ] `kubectl get externalsecret -n fourdogs-etailpet-sales-trigger` shows `Ready/True`
- [ ] k8s Secret contains both keys
- [ ] No Vault credentials in committed manifests or code
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed and merged to `fourdogs-etailpet-api-sales` branch
