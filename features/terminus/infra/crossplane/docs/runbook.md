# Crossplane Deployment Runbook
**Initiative:** terminus-infra-crossplane | **Track:** tech-change
**Environment:** terminus homelab k3s cluster
**Date:** 2026-03-29
**Author:** Amelia (Dev) via @lens batch run

---

## Overview

This runbook covers the end-to-end deployment of Crossplane on the terminus k3s cluster using
ArgoCD App-of-Apps. It includes the post-install RBAC step that cannot be automated via GitOps.

**Architecture:** ArgoCD parent Application → kustomize → crossplane-core Application (Helm) + Provider CRDs + ProviderConfig CRDs, deployed in sync-wave order.

**Target repo:** `terminus.infra` — all manifests under `apps/crossplane/`

---

## Prerequisites

- [ ] k3s cluster running and accessible (`kubectl cluster-info`)
- [ ] ArgoCD deployed and healthy (`kubectl get pods -n argocd`)
- [ ] `terminus.infra` repo PR for `feature/terminus-infra-crossplane-epic-*` merged to `main`
- [ ] ArgoCD connected to the `terminus.infra` git repo (verify via ArgoCD UI or `argocd repo list`)
- [ ] Outbound internet access from k3s nodes to `charts.crossplane.io` and `xpkg.upbound.io`

---

## Step 1 — Apply Parent Application

The `platforms/k3s/argocd/apps/crossplane.yaml` file is picked up automatically by the
root-app ArgoCD Application (which watches `platforms/k3s/argocd/apps/`). Once merged to `main`,
ArgoCD will sync it within the automated sync interval.

**Verify the parent app appears:**
```bash
kubectl get application crossplane -n argocd
# Expected: crossplane | Synced | Healthy
```

If not auto-synced, trigger manually:
```bash
argocd app sync crossplane
```

---

## Step 2 — Monitor Wave-by-Wave Progression

ArgoCD deploys resources in sync-wave order. Monitor progress:

```bash
# Watch all applications
kubectl get applications -n argocd -w

# Watch crossplane pods
kubectl get pods -n crossplane-system -w
```

**Expected progression:**
1. Wave 0: `crossplane-core` Application created → Crossplane Helm chart installs → pods running
2. Wave 1: `upbound-provider-kubernetes` and `upbound-provider-helm` applied → provider pods start
3. Wave 2: `kubernetes-default` and `helm-default` ProviderConfigs applied

**Timing:** Each wave waits for previous wave resources to be Healthy. Allow 5-10 minutes total.

---

## Step 3 — Post-Install RBAC

> ⚠️ **This step cannot be automated via GitOps.** The provider ServiceAccount name includes a
> Crossplane-generated hash and must be discovered after provider installation.

**After provider-kubernetes pod is Running and HEALTHY:**

```bash
# 1. Discover the provider ServiceAccount name
SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
  sed -e 's|serviceaccount\/|crossplane-system:|g')
echo "SA: $SA"

# 2. Verify SA exists and looks correct
# Expected output: something like crossplane-system:upbound-provider-kubernetes-abc12

# 3. Apply the ClusterRoleBinding (one of two approaches)

# Option A — kubectl imperative (fastest for one-time setup):
kubectl create clusterrolebinding provider-kubernetes-admin-binding \
  --clusterrole cluster-admin \
  --serviceaccount="${SA}" \
  --dry-run=client -o yaml | kubectl apply -f -

# Option B — patch the GitOps manifest and re-sync (preferred for drift prevention):
# 1. Get exact SA name: kubectl get sa -n crossplane-system | grep provider-kubernetes
# 2. Edit apps/crossplane/crossplane-core/rbac-provider-kubernetes.yaml
#    Update subjects[0].name to the full SA name (without namespace prefix)
# 3. Commit and push — ArgoCD will sync the updated manifest
```

**Verify the binding is working:**
```bash
kubectl get clusterrolebinding provider-kubernetes-admin-binding -o yaml
# Confirm subject SA name is correct
```

---

## Step 4 — Run Acceptance Checks

Follow the full acceptance checklist in [acceptance-results.md](acceptance-results.md).

Quick health check:
```bash
# All crossplane pods running
kubectl get pods -n crossplane-system

# Providers healthy
kubectl get providers
# Expected: HEALTHY=True for both upbound-provider-kubernetes and upbound-provider-helm

# ProviderConfigs synced
kubectl get providerconfigs.kubernetes.crossplane.io
kubectl get providerconfigs.helm.crossplane.io
# Expected: SYNCED=True for both 'default' configs

# ArgoCD applications healthy
kubectl get applications -n argocd | grep crossplane
```

---

## Step 5 — Smoke Test

Run the smoke test from [acceptance-results.md §Phase 5](acceptance-results.md).

---

## Troubleshooting

### Provider stuck in "Installing" / not HEALTHY

```bash
kubectl describe provider upbound-provider-kubernetes
# Check events for pull errors from xpkg.upbound.io
```

Common causes:
- Network: k3s nodes can't reach `xpkg.upbound.io`
- Image pull rate limit: Upbound registry rate limits unauthenticated pulls (unlikely for homelab)
- Version: If the pinned version is yanked, update `spec.package` in the provider YAML

### ProviderConfig shows SYNCED=False

Usually means the provider is not yet HEALTHY. Check provider status first:
```bash
kubectl get providers
kubectl describe providerconfig default  # for kubernetes
```

### Object managed resource fails to reconcile

1. Check the RBAC binding: see §Post-Install RBAC above
2. Verify SA name matches binding: `kubectl get clusterrolebinding provider-kubernetes-admin-binding -o yaml`
3. Restart provider pod after fixing RBAC: `kubectl delete pod -n crossplane-system -l pkg.crossplane.io/revision -l pkg.crossplane.io/resource-type=Provider`

### ArgoCD OutOfSync on cluster-scoped resources

Provider and ProviderConfig are cluster-scoped. If ArgoCD shows OutOfSync after they're applied:
- Check if `ServerSideApply=true` is set on the parent Application (it is)
- If ownership conflicts exist: `argocd app sync crossplane --server-side-apply`

### Blast-and-Repave Notes

If running a full cluster repave (k3s teardown + reinstall):
1. Delete the crossplane parent Application from ArgoCD BEFORE tearing down k3s (prevents finalizer deadlock)
   ```bash
   kubectl patch application crossplane -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge
   kubectl delete application crossplane -n argocd
   ```
2. Reinstall k3s and ArgoCD
3. Re-apply root-app: `platforms/k3s/argocd/root-app.yaml`
4. ArgoCD will re-sync crossplane from git
5. Run §Post-Install RBAC again (SA name will be a new hash after repave)

---

## Version Reference

| Component | Version | Pinned Date |
|-----------|---------|-------------|
| Crossplane Helm chart | v2.2.0 | 2026-03-29 |
| provider-kubernetes | v1.2.2 | 2026-03-29 |
| provider-helm | v1.2.1 | 2026-03-29 |

Both providers require Crossplane 2.0+. v2.2.0 satisfies this.

**Next version check:** Before next install, verify against [Upbound Marketplace](https://marketplace.upbound.io/providers/upbound/provider-kubernetes).

---

## Related Docs

- [Architecture Decision Document](architecture.md)
- [Acceptance Criteria Checklist](acceptance-results.md)
- [Adversarial Review Report](adversarial-review-report.md)
- [Epics and Stories](epics.md)
