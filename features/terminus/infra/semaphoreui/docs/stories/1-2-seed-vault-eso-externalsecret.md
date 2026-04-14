# Story 1.2: Seed Vault KV and Create ESO ExternalSecret

Status: ready-for-dev

## Story

As a platform operator,
I want Semaphore UI's credentials stored in Vault and synced to a Kubernetes Secret via ESO,
So that the Semaphore Deployment can read all required credentials without any plaintext secrets in manifests.

## Acceptance Criteria

1. **Given** Vault is operational at `vault.trantor.internal:8200` with KV v2 at `secret/`
   **When** `secret/terminus/infra/semaphoreui` is seeded with 5 fields AND the ESO ExternalSecret manifest is applied
   **Then** k8s Secret `semaphoreui-secrets` exists in the `terminus-infra` namespace
2. The Secret contains exactly 5 keys: `SEMAPHORE_ADMIN`, `SEMAPHORE_ADMIN_PASSWORD`, `SEMAPHORE_ADMIN_EMAIL`, `SEMAPHORE_DB_PASS`, `SEMAPHORE_ACCESS_KEY_ENCRYPTION`
3. `kubectl get externalsecret semaphoreui-externalsecret -n terminus-infra` shows `STATUS: SecretSynced`
4. No vault credentials or actual secret values appear in any committed manifest

## Tasks

- [ ] Task 1: Seed Vault KV (operator manual step — NOT committed to git)
  - [ ] Generate a random 32+ character string for `access_key_encryption`: `openssl rand -base64 32`
  - [ ] Choose `db_password` value (must match the value used in Story 1.1 `tofu apply`)
  - [ ] Run: `vault kv put secret/terminus/infra/semaphoreui SEMAPHORE_ADMIN=<admin_user> SEMAPHORE_ADMIN_PASSWORD=<admin_password> SEMAPHORE_ADMIN_EMAIL=<admin_email> SEMAPHORE_DB_PASS=<db_password> SEMAPHORE_ACCESS_KEY_ENCRYPTION=<encryption_key>`
  - [ ] Verify: `vault kv get secret/terminus/infra/semaphoreui` — confirm all 5 keys present
- [ ] Task 2: Create `externalsecret.yaml` manifest in `terminus.infra`
  - [ ] File path: `platforms/k3s/manifests/terminus-infra/semaphoreui/externalsecret.yaml`
  - [ ] `apiVersion: external-secrets.io/v1beta1`, `kind: ExternalSecret`
  - [ ] `metadata.name: semaphoreui-externalsecret`, `namespace: terminus-infra`
  - [ ] `spec.secretStoreRef.name: vault-backend`, `spec.secretStoreRef.kind: ClusterSecretStore`
  - [ ] `spec.target.name: semaphoreui-secrets`
  - [ ] `spec.refreshInterval: 15m`
  - [ ] `spec.data[]` — map all 5 Vault fields to the `SEMAPHORE_*` key names
  - [ ] Create the `semaphoreui/` manifest directory if it doesn't exist
- [ ] Task 3: Verify ExternalSecret sync
  - [ ] Apply manifest: `kubectl apply -f platforms/k3s/manifests/terminus-infra/semaphoreui/externalsecret.yaml`
  - [ ] Watch: `kubectl get externalsecret semaphoreui-externalsecret -n terminus-infra -w` — wait for `SecretSynced`
  - [ ] Verify Secret keys: `kubectl get secret semaphoreui-secrets -n terminus-infra -o jsonpath='{.data}' | python3 -m json.tool` — confirm 5 keys present

## Dev Notes

### ExternalSecret Field Mapping

Vault KV field names (keys in `vault kv put`) must map to the Kubernetes Secret key names (`SEMAPHORE_*`). Use the following mapping:

```yaml
spec:
  data:
    - secretKey: SEMAPHORE_ADMIN
      remoteRef:
        key: secret/terminus/infra/semaphoreui
        property: SEMAPHORE_ADMIN
    - secretKey: SEMAPHORE_ADMIN_PASSWORD
      remoteRef:
        key: secret/terminus/infra/semaphoreui
        property: SEMAPHORE_ADMIN_PASSWORD
    - secretKey: SEMAPHORE_ADMIN_EMAIL
      remoteRef:
        key: secret/terminus/infra/semaphoreui
        property: SEMAPHORE_ADMIN_EMAIL
    - secretKey: SEMAPHORE_DB_PASS
      remoteRef:
        key: secret/terminus/infra/semaphoreui
        property: SEMAPHORE_DB_PASS
    - secretKey: SEMAPHORE_ACCESS_KEY_ENCRYPTION
      remoteRef:
        key: secret/terminus/infra/semaphoreui
        property: SEMAPHORE_ACCESS_KEY_ENCRYPTION
```

### Password Consistency

- The `SEMAPHORE_DB_PASS` value in Vault **MUST** be identical to the `db_password` used in Story 1.1 `tofu apply`. Seed Vault before running tofu apply (or ensure consistency).

### ESO ClusterSecretStore

- `vault-backend` ClusterSecretStore already exists from the k3s platform setup. Do NOT create a new one.
- Verify it's healthy before applying: `kubectl get clustersecretstore vault-backend`

### Vault Path Convention

- Vault path is `secret/terminus/infra/semaphoreui` — note this uses `semaphoreui` (not `semaphore`, which is the DB name). This is intentional and specified in TD-008.

### Security Notes

- `SEMAPHORE_ACCESS_KEY_ENCRYPTION` is used by Semaphore to encrypt stored Ansible access keys at rest. It must be at least 32 characters. If ever changed, all previously stored Semaphore tasks/keys become unreadable — do NOT rotate unless intentional.
- Never commit actual credential values. Task 1 is operator-manual and not git-tracked.

### Architecture Reference

- [Source: docs/terminus/infra/semaphoreui/architecture.md — ExternalSecret Configuration section]
- [Source: docs/terminus/infra/semaphoreui/tech-decisions.md — TD-008 (Vault path), TD-009 (ESO ExternalSecret)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

### Completion Notes List

### File List

- `platforms/k3s/manifests/terminus-infra/semaphoreui/externalsecret.yaml` (new, in `terminus.infra`)
