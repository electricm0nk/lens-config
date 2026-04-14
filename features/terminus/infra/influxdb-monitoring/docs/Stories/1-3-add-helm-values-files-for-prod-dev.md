# Story 1.3: Add Helm Values Files for Prod/Dev

Status: done

**Epic:** 1 — GitOps InfluxDB Deployment Foundation
**Initiative:** terminus-infra-influxdb-monitoring
**Date:** 2026-04-12
**Implemented:** d1e6652 (terminus.infra)

---

## Story

As an infrastructure operator,
I want environment-specific Helm values files for InfluxDB prod and dev,
so that each environment has appropriately sized resources and isolated configuration.

---

## Context

**Repo:** `terminus.infra`
**Files:**
- `platforms/k3s/helm/influxdb/values.yaml` (prod)
- `platforms/k3s/helm/influxdb/values-dev.yaml` (dev)

---

## Technical Notes

**Prod values (`values.yaml`):**
- `persistence.size: 50Gi`
- `retention_policy: 30d`
- Memory: requests 512Mi / limits 2Gi
- CPU: requests 200m / limits 1000m
- `organization: terminus`
- `bucket: monitoring`
- Admin credentials: referenced via Vault/ESO (bootstrap pending — see Epic 2)

**Dev values (`values-dev.yaml`):**
- `persistence.size: 10Gi`
- `retention_policy: 7d`
- Memory: requests 256Mi / limits 1Gi
- CPU: requests 100m / limits 500m
- `organization: terminus-dev`
- `bucket: monitoring-dev`

---

## Acceptance Criteria

1. `platforms/k3s/helm/influxdb/values.yaml` exists with prod-appropriate resource sizing
2. `platforms/k3s/helm/influxdb/values-dev.yaml` exists with dev-appropriate reduced sizing
3. No credentials are present in either values file (bootstrap auth is secret-managed — Epic 2)
4. Both files are referenced correctly by their respective ArgoCD Application manifests

## Implementation Notes

Implemented in commit `d1e6652` in `TargetProjects/terminus/infra/terminus.infra`. No further action required.
