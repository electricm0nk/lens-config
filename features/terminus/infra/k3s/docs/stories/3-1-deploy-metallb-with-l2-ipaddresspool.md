# Story 3.1: Deploy MetalLB with L2 IPAddressPool

## Status: done

## Story

As an operator,
I want MetalLB deployed in L2 mode with the `10.0.0.126–10.0.0.254` IP pool,
So that Kubernetes `LoadBalancer` services can receive routable IPs on the local network.

## Acceptance Criteria

- **Given** ArgoCD App-of-Apps managing cluster from Epic 2 and the `metallb-system` namespace present
- **When** the MetalLB `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/metallb.yaml`
- **Then** all MetalLB pods (controller + speaker) are `Running` in `metallb-system`
- **And** an `IPAddressPool` resource exists for `10.0.0.126–10.0.0.254`
- **And** an `L2Advertisement` resource is present referencing that pool
- **And** MetalLB Helm values are sourced exclusively from `infra/k3s/helm/metallb/values.yaml`
- **And** all MetalLB resources carry the 4 standard labels

## Tasks / Subtasks

- [x] Task 1: Create MetalLB Helm values
  - [x] `infra/k3s/helm/metallb/values.yaml` — no CP tolerations, FRR disabled (L2 mode)
- [x] Task 2: Create MetalLB custom resource manifests
  - [x] `infra/k3s/manifests/metallb/ipaddresspool.yaml` — `IPAddressPool` terminus-pool for `10.0.0.126-10.0.0.254`
  - [x] `infra/k3s/manifests/metallb/l2advertisement.yaml` — `L2Advertisement` terminus-l2 referencing pool
  - [x] Both carry 4 standard labels
- [x] Task 3: Update metallb.yaml ArgoCD Application
  - [x] `infra/k3s/argocd/apps/metallb.yaml` — 3-source: chart + values ref + manifests dir
  - [x] `syncPolicy.automated.prune: true, selfHeal: true`

## Dev Notes

**MetalLB chart source:** `https://metallb.github.io/metallb`

**IPAddressPool:**
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: terminus-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.126-10.0.0.254
```

**L2Advertisement:**
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: terminus-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - terminus-pool
```

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- 3-source ArgoCD App includes manifests dir alongside chart; ArgoCD retries CRD manifests until MetalLB chart installs CRDs — acceptable GitOps pattern
- FRR disabled: L2 mode uses ARP/NDP only, no BGP routing daemon needed
- PR: electricm0nk/terminus.infra#34
### Change List
- `infra/k3s/helm/metallb/values.yaml` — FRR disabled, no CP tolerations
- `infra/k3s/manifests/metallb/ipaddresspool.yaml` — IPAddressPool terminus-pool 10.0.0.126-10.0.0.254
- `infra/k3s/manifests/metallb/l2advertisement.yaml` — L2Advertisement terminus-l2
- `infra/k3s/argocd/apps/metallb.yaml` — full 3-source Application
