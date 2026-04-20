# Story: transactions-2-02

## Context

**Feature:** transactions
**Sprint:** 2 — Deploy & Validate
**Priority:** high
**Estimate:** S

## User Story

As an operator, I want the eTailPet API credential provisioned in Vault and synced to k8s via ESO,
so that the trigger worker has secrets injected at runtime without any credential in code or Helm values.

## Acceptance Criteria

- [ ] Vault secret exists at `secret/terminus/fourdogs/etailpet-api` with correct field(s):
  - `ETAILPET_API_KEY` (or auth-type equivalent per transactions-0-01)
  - `ETAILPET_TRIGGER_URL` — the eTailPet trigger endpoint URL
- [ ] ESO `ExternalSecret` manifest at `deploy/helm/fourdogs-etailpet-trigger/templates/external-secret.yaml`
- [ ] ExternalSecret targets k8s secret name `fourdogs-etailpet-api-secrets` in namespace `fourdogs-central`
- [ ] ExternalSecret uses `ClusterSecretStore/vault-backend` (consistent with all other fourdogs services)
- [ ] ESO sync confirmed: `kubectl get secret fourdogs-etailpet-api-secrets -n fourdogs-central -o yaml` shows `data` populated
- [ ] No Vault credential or field value appears in any committed file, PR diff, or log output

## Technical Notes

**ExternalSecret spec template:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: fourdogs-etailpet-api-secrets
  namespace: fourdogs-central
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: fourdogs-etailpet-api-secrets
    creationPolicy: Owner
  data:
    - secretKey: ETAILPET_API_KEY
      remoteRef:
        key: secret/terminus/fourdogs/etailpet-api
        property: ETAILPET_API_KEY
    - secretKey: ETAILPET_TRIGGER_URL
      remoteRef:
        key: secret/terminus/fourdogs/etailpet-api
        property: ETAILPET_TRIGGER_URL
```

Adjust field names based on transactions-0-01 spike findings (e.g. `ETAILPET_USERNAME` +
`ETAILPET_PASSWORD` if basic auth).

**Vault provisioning** (run by operator with Vault admin token — not in CI):
```bash
vault kv put secret/terminus/fourdogs/etailpet-api \
  ETAILPET_API_KEY="<key>" \
  ETAILPET_TRIGGER_URL="<url>"
```

**Verification after ESO sync:**
```bash
kubectl get externalsecret fourdogs-etailpet-api-secrets -n fourdogs-central
# STATUS should be: SecretSynced / True
```

## Dependencies

**Blocked by:** transactions-0-01 (Vault path and field names depend on spike findings),
transactions-2-01 (namespace must exist before ESO syncs to it)
**Blocks:** transactions-2-03 (live trigger requires credentials injected)

## Definition of Done

- [ ] Code compiles with no errors or warnings (`go build ./...`)
- [ ] `go test ./...` passes
- [ ] `go vet ./...` passes
- [ ] No credentials, tokens, or secrets in code or committed files
- [ ] Structured log events fire correctly for the story's scope
- [ ] Story acceptance criteria verified by author
- [ ] PR reviewed (or self-reviewed for solo work) and merged to `transactions` branch

## Open Questions

None — field names resolved by transactions-0-01.
