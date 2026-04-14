# Story 1.1: Add `influxdb` ArgoCD Application Manifest (Prod)

Status: done

**Epic:** 1 — GitOps InfluxDB Deployment Foundation
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Implemented:** d1e6652 (terminus.infra)

---

## Story

As an infrastructure operator,
I want an ArgoCD Application manifest for InfluxDB targeting the `monitoring` namespace,
so that ArgoCD manages the InfluxDB prod deployment via GitOps with auto-sync.

---

## Context

**Repo:** `terminus.infra`
**File:** `platforms/k3s/argocd/apps/influxdb.yaml`
**Namespace:** `monitoring`
**Pattern:** Multi-source ArgoCD app (chart source + values ref source) — same pattern as traefik and cert-manager.

---

## Technical Notes

- Chart: `influxdb2` from `https://helm.influxdata.com/` at `targetRevision: 2.*`
- Values source: `terminus.infra` main → `platforms/k3s/helm/influxdb/values.yaml`
- Auto-sync, prune, and self-heal enabled
- `CreateNamespace=true` via syncOptions
- `releaseName: influxdb`
- Destination: `in-cluster` / `monitoring` namespace

---

## Acceptance Criteria

1. `platforms/k3s/argocd/apps/influxdb.yaml` exists in terminus.infra
2. ArgoCD Application uses multi-source pattern referencing the influxdata Helm chart
3. Values file is sourced from terminus.infra repo at `platforms/k3s/helm/influxdb/values.yaml`
4. `monitoring` namespace is auto-created on first sync (`CreateNamespace=true`)
5. Auto-sync, prune, and self-heal are enabled

## Implementation Notes

Implemented in commit `d1e6652` in `TargetProjects/terminus/infra/terminus.infra`. No further action required.
