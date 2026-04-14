# Story 2.3: Validate InjectedIdentity RBAC for provider-kubernetes

Status: ready-for-dev

## Story

As a platform engineer,
I want to verify and apply the ClusterRoleBinding required for `provider-kubernetes` InjectedIdentity mode,
so that Crossplane's Kubernetes provider can manage Objects across namespaces without permission errors.

## Context

`InjectedIdentity` for `provider-kubernetes` is **not zero-setup**. The provider pod runs
as the Crossplane ServiceAccount, which does not have cross-namespace object management
permissions by default. Without an explicit ClusterRoleBinding, all `Object` managed resources
will fail to reconcile with `Forbidden` errors.

This was flagged in adversarial finding #6 and pre-confirmed by Upbound Marketplace docs
(documented in Story 1.1 and architecture.md `## RBAC Notes`).

**Depends-on:** Story 2.1 (provider-kubernetes deployed and pod running)

### Key Distinction from Story 2.2

`provider-helm` does NOT need this ClusterRoleBinding. Only `provider-kubernetes` requires it.

## Acceptance Criteria

1. Crossplane provider-kubernetes pod is Running: `kubectl get pods -n crossplane-system | grep provider-kubernetes`
2. Provider HEALTHY: `kubectl get providers provider-kubernetes` shows `HEALTHY=True`
3. ClusterRoleBinding `provider-kubernetes-admin-binding` exists: `kubectl get clusterrolebinding provider-kubernetes-admin-binding`
4. Smoke test: create a test `Object` managed resource in default namespace and verify it reconciles to `SYNCED=True`
5. Smoke test `Object` manifest is removed after verification (do not leave test resources in cluster)
6. RBAC finding documented in architecture.md `## RBAC Notes` is confirmed accurate (or corrected if incorrect)
7. Finding: confirm if `provider-helm` requires any additional RBAC (expected: no — document the finding)

## Tasks / Subtasks

- [ ] Task 1: Verify provider-kubernetes pod is Running (AC: #1–#2)
  - [ ] `kubectl get pods -n crossplane-system`
  - [ ] `kubectl get providers` — confirm provider-kubernetes shows `HEALTHY=True`
  - [ ] If not healthy: check pod logs `kubectl logs -n crossplane-system <pod>` and resolve before continuing

- [ ] Task 2: Apply ClusterRoleBinding (AC: #3)
  - [ ] Run the binding command from architecture.md `## RBAC Notes`:
    ```bash
    SA=$(kubectl -n crossplane-system get sa -o name | grep provider-kubernetes | \
      sed -e 's|serviceaccount\/|crossplane-system:|g')
    kubectl create clusterrolebinding provider-kubernetes-admin-binding \
      --clusterrole cluster-admin \
      --serviceaccount="${SA}"
    ```
  - [ ] Verify: `kubectl get clusterrolebinding provider-kubernetes-admin-binding`

- [ ] Task 3: Smoke test with test Object (AC: #4–#5)
  - [ ] Create a minimal test Object manifest (see Dev Notes for template)
  - [ ] Apply: `kubectl apply -f test-object.yaml`
  - [ ] Wait for reconciliation: `kubectl get object test-configmap`
  - [ ] Verify SYNCED=True and check ConfigMap exists in default namespace
  - [ ] Clean up: `kubectl delete -f test-object.yaml` then `kubectl delete configmap test-crossplane-object`

- [ ] Task 4: Verify provider-helm RBAC (AC: #7)
  - [ ] Confirm provider-helm pod is Running and HEALTHY
  - [ ] No additional ClusterRoleBinding needed (expected) — document finding in story completion notes

## Dev Notes

### Why cluster-admin for Homelab

Homelab context: RBAC hardening is explicitly deferred to a future security initiative
(per architecture.md `## Core Architectural Decisions — Decision 3`). Using `cluster-admin`
is acceptable for phase 1. A production implementation would scope permissions to the
specific namespaces and resources the provider manages.

### Test Object Template

```yaml
# File: test-object.yaml (temporary — delete after smoke test)
apiVersion: kubernetes.crossplane.io/v1alpha2
kind: Object
metadata:
  name: test-configmap
spec:
  forProvider:
    manifest:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: test-crossplane-object
        namespace: default
      data:
        test: "crossplane-provider-kubernetes-smoke-test"
  providerConfigRef:
    name: default
```

> **Warning:** This Object requires the `kubernetes-default` ProviderConfig (Story 3.1) to exist.
> If Story 3.1 is not complete, the Object will fail with `ProviderConfig not found`.
> In that case, skip Task 3 and complete Story 3.1 first, then return to run the smoke test.

### If Provider-Kubernetes Fails to Start

Check:
1. `kubectl describe provider provider-kubernetes` — look for `Installed` and `Healthy` conditions
2. Pod logs: `kubectl logs -n crossplane-system deployment/provider-kubernetes-*`
3. Common issue: pulling from `xpkg.upbound.io` blocked by network policy — check egress rules

### References

- [Source: docs/terminus/infra/crossplane/architecture.md — RBAC Notes, Decision 3]
- [Source: docs/terminus/infra/crossplane/adversarial-review-report.md — Finding #6]
- [Source: docs/implementation-artifacts/1-1-research-and-pin-crossplane-provider-versions.md — RBAC pre-research]
- [Upstream: https://marketplace.upbound.io/providers/upbound/provider-kubernetes/v1.2.2#required-configuration]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Completion Notes List

_To be filled in by dev agent during implementation_

### File List

- No files created — this is a cluster validation story
- `terminus.infra/apps/crossplane/crossplane-core/rbac-provider-kubernetes.yaml` — create only if default ClusterRole insufficient
