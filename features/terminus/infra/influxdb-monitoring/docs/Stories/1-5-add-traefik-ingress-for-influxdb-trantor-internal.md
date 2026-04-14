# Story 1.5: Add Traefik Ingress for influxdb.trantor.internal (Prod + Dev)

Status: done

**Epic:** 1 — GitOps InfluxDB Deployment Foundation
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Depends on:** Stories 1.1, 1.2
**Required by:** Stories 3.1, 3.2 (verify playbooks use external DNS)
**Implemented:** this session

---

## Story

As an infrastructure operator,
I want Traefik ingress rules for `influxdb.trantor.internal` and `influxdb-dev.trantor.internal`,
so that the InfluxDB UI and `/health` endpoint are reachable externally for operator access and verify playbooks.

---

## Context

**Repo:** `terminus.infra`
**Namespaces:** `monitoring` (prod), `monitoring-dev` (dev)
**VIP:** `10.0.0.126` (MetalLB — wildcard `*.trantor.internal` default, no NAS A-record needed)
**IngressClass:** `traefik`
**InfluxDB service port:** `8086`

---

## Technical Notes

**Prod ingress — service name:** `influxdb` (Helm releaseName = `influxdb`, port 8086)
**Dev ingress — service name:** `influxdb-dev` (releaseName = `influxdb-dev`, port 8086)

Files created:
- `platforms/k3s/k8s/influxdb-ingress/ingress.yaml`
- `platforms/k3s/k8s/influxdb-dev-ingress/ingress.yaml`
- `platforms/k3s/argocd/apps/influxdb-ingress.yaml`
- `platforms/k3s/argocd/apps/influxdb-dev-ingress.yaml`

CoreDNS configmap and `/etc/hosts` on `vault.trantor.internal` are still required for Semaphore runner resolution.

---

## Acceptance Criteria

1. `influxdb-ingress` ArgoCD app exists (wave 3, namespace `monitoring`)
2. `influxdb-dev-ingress` ArgoCD app exists (wave 3, namespace `monitoring-dev`)
3. Both ingress rules use `ingressClassName: traefik`
4. `curl http://influxdb.trantor.internal/health` returns `{"status":"pass"}` from vault.trantor.internal
5. `curl http://influxdb-dev.trantor.internal/health` returns `{"status":"pass"}` from vault.trantor.internal
