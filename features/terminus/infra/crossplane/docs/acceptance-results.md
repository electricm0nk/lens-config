# Crossplane Post-Install Acceptance Criteria
**Initiative:** terminus-infra-crossplane | **Story 3.4**
**Date:** 2026-03-29 (pre-install — this document defines the acceptance procedure)

---

## Acceptance Checklist

Complete each check in order. Record actual results in the **Actual** column.

### Phase 1 — Crossplane Core Health

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 1.1 | Crossplane pods running | `kubectl get pods -n crossplane-system` | All pods `Running` | |
| 1.2 | Crossplane CRDs registered | `kubectl get crds \| grep crossplane.io` | Provider, ProviderConfig, Object, Release CRDs present | |
| 1.3 | ArgoCD crossplane-core synced | `kubectl get application crossplane-core -n argocd` | `Synced / Healthy` | |

### Phase 2 — Provider Health

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 2.1 | provider-kubernetes installed | `kubectl get providers` | `upbound-provider-kubernetes` listed | |
| 2.2 | provider-kubernetes HEALTHY | `kubectl get providers upbound-provider-kubernetes` | `HEALTHY=True, INSTALLED=True` | |
| 2.3 | provider-helm installed | `kubectl get providers` | `upbound-provider-helm` listed | |
| 2.4 | provider-helm HEALTHY | `kubectl get providers upbound-provider-helm` | `HEALTHY=True, INSTALLED=True` | |
| 2.5 | Provider pods running | `kubectl get pods -n crossplane-system \| grep provider` | Both provider pods `Running` | |

### Phase 3 — RBAC Validation (provider-kubernetes)

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 3.1 | Provider SA exists | `kubectl get sa -n crossplane-system \| grep provider-kubernetes` | SA present | |
| 3.2 | ClusterRoleBinding applied | `kubectl get clusterrolebinding provider-kubernetes-admin-binding` | Binding exists | |
| 3.3 | SA name in binding correct | `kubectl get clusterrolebinding provider-kubernetes-admin-binding -o yaml` | Subject SA name matches actual SA | |

> ⚠️ If step 3.2 fails: the SA name in `rbac-provider-kubernetes.yaml` may need updating.
> Run the discovery command from the runbook §Post-Install RBAC and patch the manifest.

### Phase 4 — ProviderConfig Health

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 4.1 | kubernetes ProviderConfig synced | `kubectl get providerconfigs.kubernetes.crossplane.io` | `default` present, `SYNCED=True` | |
| 4.2 | helm ProviderConfig synced | `kubectl get providerconfigs.helm.crossplane.io` | `default` present, `SYNCED=True` | |

### Phase 5 — End-to-End Smoke Test

Apply a test `Object` managed resource and verify it reconciles successfully:

```yaml
# test-object.yaml — temporary smoke test, delete after validation
apiVersion: kubernetes.crossplane.io/v1alpha2
kind: Object
metadata:
  name: crossplane-smoke-test
spec:
  providerConfigRef:
    name: default
  forProvider:
    manifest:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: crossplane-smoke-test
        namespace: default
      data:
        test: "crossplane-provider-kubernetes-smoke-test"
```

```bash
kubectl apply -f test-object.yaml
kubectl get object crossplane-smoke-test
# Expected: SYNCED=True, READY=True
kubectl get configmap crossplane-smoke-test -n default
# Expected: ConfigMap exists with test data

# Cleanup
kubectl delete -f test-object.yaml
kubectl delete configmap crossplane-smoke-test -n default
```

### Phase 6 — ArgoCD Application Status

| # | Check | Command | Expected | Actual |
|---|-------|---------|----------|--------|
| 6.1 | Parent crossplane app | `kubectl get application crossplane -n argocd` | `Synced / Healthy` | |
| 6.2 | crossplane-core sub-app | `kubectl get application crossplane-core -n argocd` | `Synced / Healthy` | |

---

## Known Issues / Runbook References

- **SA name mismatch (RBAC):** See [runbook.md §Post-Install RBAC](runbook.md#post-install-rbac)
- **Provider HEALTHY timeout:** Providers can take 2-5 minutes to pull and install. Wait before declaring failure.
- **ProviderConfig SYNCED=False:** Usually means provider is not yet HEALTHY. Check provider status first.
