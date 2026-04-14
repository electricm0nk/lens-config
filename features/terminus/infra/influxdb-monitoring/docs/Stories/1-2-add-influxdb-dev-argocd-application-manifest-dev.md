# Story 1.2: Add `influxdb-dev` ArgoCD Application Manifest (Dev)

Status: done

**Epic:** 1 — GitOps InfluxDB Deployment Foundation
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Implemented:** d1e6652 (terminus.infra)

---

## Story

As an infrastructure operator,
I want a separate ArgoCD Application manifest for InfluxDB targeting the `monitoring-dev` namespace,
so that the dev environment has an isolated, independently managed InfluxDB instance.

---

## Context

**Repo:** `terminus.infra`
**File:** `platforms/k3s/argocd/apps/influxdb-dev.yaml`
**Namespace:** `monitoring-dev`
**Pattern:** Same multi-source pattern as prod; references dev-specific values file.

---

## Technical Notes

- Same chart source as prod: `influxdb2` from `https://helm.influxdata.com/`
- Values source: `terminus.infra` main → `platforms/k3s/helm/influxdb/values-dev.yaml`
- Dev-sizing: 10Gi persistence, 7d retention, 256Mi/1Gi mem, 100m/500m cpu
- `organization: terminus-dev`, `bucket: monitoring-dev`
- `releaseName: influxdb-dev`
- Destination: `in-cluster` / `monitoring-dev` namespace

---

## Acceptance Criteria

1. `platforms/k3s/argocd/apps/influxdb-dev.yaml` exists in terminus.infra
2. References `values-dev.yaml` for environment-appropriate sizing
3. `monitoring-dev` namespace is auto-created (`CreateNamespace=true`)
4. Dev app does not affect the prod `monitoring` namespace
5. Auto-sync enabled with prune and self-heal

## Implementation Notes

Implemented in commit `d1e6652` in `TargetProjects/terminus/infra/terminus.infra`. No further action required.
