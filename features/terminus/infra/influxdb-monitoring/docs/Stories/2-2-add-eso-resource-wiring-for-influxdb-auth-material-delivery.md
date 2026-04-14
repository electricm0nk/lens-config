# Story 2.2: Add ESO Resource Wiring for InfluxDB Auth Material Delivery

Status: done

**Epic:** 2 — Secrets and Bootstrap Hardening
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Story 2.1
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want ExternalSecret resources that pull InfluxDB admin credentials from Vault and deliver them as k8s Secrets,
so that the InfluxDB Helm chart can reference the admin credentials without any secrets in git.

---

## Context

**Repo:** `terminus.infra`
**ESO:** ExternalSecrets Operator already deployed in cluster
**Vault:** KV v2 at paths defined in Story 2.1
**Pattern:** Follows existing ESO ExternalSecret manifests used for other terminus services

---

## Technical Notes

**ExternalSecret manifest location:**
- Prod: `platforms/k3s/helm/influxdb/external-secret.yaml` (or namespaced under argocd app resources)
- Dev: `platforms/k3s/helm/influxdb/external-secret-dev.yaml`

**Prod ExternalSecret structure:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: influxdb-admin
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: influxdb-admin
    creationPolicy: Owner
  data:
    - secretKey: admin-password
      remoteRef:
        key: terminus/monitoring/influxdb/admin
        property: password
    - secretKey: admin-token
      remoteRef:
        key: terminus/monitoring/influxdb/admin
        property: token
```

**Helm values update required:**
Add to `values.yaml`:
```yaml
adminUser:
  existingSecret: influxdb-admin
  existingSecretPasswordKey: admin-password
  existingSecretTokenKey: admin-token
```

**Delivery approach:**
- ExternalSecret manifests committed to terminus.infra under the influxdb Helm values path
- Wire as an additional source in the ArgoCD multi-source app config (or as a separate `-infra` ArgoCD Application)
- ESO manifests are plain k8s manifests, not Helm-managed

**Check existing ESO ClusterSecretStore name** — look at other ExternalSecret manifests in the repo to confirm the correct `secretStoreRef.name`.

**REQUIRED — `ignoreDifferences` pattern (cicd.md mandate):**
ESO injects default fields that ArgoCD does not track, causing persistent `OutOfSync`. Every ArgoCD Application manifest that contains an ExternalSecret MUST include `ignoreDifferences`. The InfluxDB admin secret has 2 data entries (indices 0 and 1):
```yaml
ignoredDifferences:
  - group: external-secrets.io
    kind: ExternalSecret
    jsonPointers:
      - /spec/data/0/remoteRef/conversionStrategy
      - /spec/data/0/remoteRef/decodingStrategy
      - /spec/data/0/remoteRef/metadataPolicy
      - /spec/data/1/remoteRef/conversionStrategy
      - /spec/data/1/remoteRef/decodingStrategy
      - /spec/data/1/remoteRef/metadataPolicy
      - /spec/target/deletionPolicy
      - /spec/target/template/engineVersion
      - /spec/target/template/mergePolicy
```
**After adding ignoreDifferences:** sync the root app (`terminus-infra-k3s-root`) FIRST, then re-sync the child app. ArgoCD will not pick up `ignoreDifferences` until the root app reconciles.

**Vault KV path must exist BEFORE ESO syncs** (cicd.md mandate): write Vault credentials (Story 2.1) before the first ArgoCD sync. ESO immediately enters `SecretSyncedError` if the path is absent.

---

## Acceptance Criteria

1. `ExternalSecret` manifest exists for prod namespace `monitoring`
2. `ExternalSecret` manifest exists for dev namespace `monitoring-dev`
3. Both manifests reference the correct Vault paths from Story 2.1
4. Both manifests use the correct `ClusterSecretStore` name
5. `values.yaml` and `values-dev.yaml` updated to reference `adminUser.existingSecret`
6. No credentials in git
7. After cluster apply: `kubectl get secret influxdb-admin -n monitoring` returns a Secret

## Tasks / Subtasks

- [ ] Task 1: Check existing ESO ClusterSecretStore name in repo
  - [ ] Find existing ExternalSecret or VaultAuth manifests in terminus.infra
  - [ ] Confirm `secretStoreRef.name` and `secretStoreRef.kind`

- [ ] Task 2: Create ESO ExternalSecret for prod
  - [ ] Create `platforms/k3s/helm/influxdb/external-secret.yaml`
  - [ ] Verify Vault path matches Story 2.1 definitions

- [ ] Task 3: Create ESO ExternalSecret for dev
  - [ ] Create `platforms/k3s/helm/influxdb/external-secret-dev.yaml`
  - [ ] Verify dev Vault path matches Story 2.1 definitions

- [ ] Task 4: Update Helm values to reference existing secrets
  - [ ] Update `values.yaml`: add `adminUser.existingSecret` block
  - [ ] Update `values-dev.yaml`: add `adminUser.existingSecret` block

- [ ] Task 5: Wire ExternalSecret manifests into ArgoCD app source
  - [ ] Decide: additional source in influxdb.yaml OR standalone namespace apply
  - [ ] Update ArgoCD app manifests if needed

- [ ] Task 6: Commit and verify
  - [ ] `git commit -m "feat(monitoring): add ESO wiring for influxdb admin credentials"`
  - [ ] Verify `kubectl get secret influxdb-admin -n monitoring` returns Secret after sync
