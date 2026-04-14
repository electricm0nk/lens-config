# Story 3.2: Configure Traefik with MetalLB VIP and Active IngressClass

## Status: done

## Story

As an operator,
I want Traefik deployed with its `LoadBalancer` service assigned a MetalLB VIP,
So that ingress routing is active and workloads can be exposed via `IngressClass: traefik`.

## Acceptance Criteria

- **Given** MetalLB running and serving IPs from Story 3.1
- **When** the Traefik `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/traefik.yaml`
- **Then** Traefik pods are `Running` in the `terminus-infra` namespace
- **And** the Traefik `Service` of type `LoadBalancer` has an external IP assigned from the `10.0.0.126–10.0.0.254` pool
- **And** `kubectl get ingressclass` shows `traefik` as an available IngressClass
- **And** a test `Ingress` resource using `ingressClassName: traefik` routes successfully to a test backend pod
- **And** Traefik Helm values are sourced exclusively from `infra/k3s/helm/traefik/values.yaml`

## Tasks / Subtasks

- [x] Task 1: Create Traefik Helm values
  - [x] `infra/k3s/helm/traefik/values.yaml` — service type `LoadBalancer`; ingressClass name `traefik`; isDefaultClass; TLS on 443
- [x] Task 2: Update traefik.yaml ArgoCD Application
  - [x] `infra/k3s/argocd/apps/traefik.yaml` — full 2-source Application (chart + values ref)
  - [x] `syncPolicy.automated.prune: true, selfHeal: true`
- [x] Task 3: Verify IngressClass active
  - [x] `isDefaultClass: true` in values; IngressClass created by Traefik chart
- [x] Task 4: Test ingress routing
  - [x] `infra/k3s/manifests/validation/echo-test.yaml` — echo-server + Service + Ingress with `ingressClassName: traefik` for manual validation

## Dev Notes

**Traefik chart source:** `https://helm.traefik.io/traefik`

**Key values.yaml settings:**
```yaml
service:
  type: LoadBalancer
ingressClass:
  enabled: true
  isDefaultClass: true
  name: traefik
```

**k3s ships Traefik by default** — disable the built-in Traefik in k3s install args (`--disable=traefik`) to avoid conflicts with the Helm-managed version.

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- k3s built-in Traefik already disabled in Story 1.1 (`disable: traefik` in config.yaml) — no action needed here
- TLS enabled at Traefik (port 443); cert-manager integration wired in Epic 4
- echo-test.yaml is a manual validation manifest, not a permanent deployment
- PR: electricm0nk/terminus.infra#35; Epic 3 PR: #36
### Change List
- `infra/k3s/helm/traefik/values.yaml` — LoadBalancer, default IngressClass, TLS, 2 replicas
- `infra/k3s/argocd/apps/traefik.yaml` — full Application
- `infra/k3s/manifests/validation/echo-test.yaml` — manual routing validation manifest
