# terminus-platform-grafana Architecture

## Initiative

**Track:** tech-change
**Domain:** terminus
**Service:** platform
**Goal:** Deploy Grafana on k3s via ArgoCD as a GitOps-managed observability dashboard, wired to InfluxDB and kube-metrics (Prometheus), with a starter InfluxDB dashboard provisioned declaratively.

---

## Deployment

- **Helm chart:** `grafana/grafana` from `https://grafana.github.io/helm-charts`
- **Repo:** `TargetProjects/terminus/infra/terminus.infra`
- **ArgoCD apps:**
  - `platforms/k3s/argocd/apps/grafana.yaml` (prod)
  - `platforms/k3s/argocd/apps/grafana-dev.yaml` (dev)
- **Helm values:**
  - `platforms/k3s/helm/grafana/values.yaml` (prod)
  - `platforms/k3s/helm/grafana/values-dev.yaml` (dev)

## Environments

| Env | Namespace | Hostname | Notes |
|-----|-----------|----------|-------|
| Prod | `monitoring` | `grafana.trantor.internal` | Shared namespace with InfluxDB |
| Dev | `monitoring-dev` | `grafana-dev.trantor.internal` | Shared namespace with InfluxDB dev |

---

## Data Sources

### 1. InfluxDB
- **Type:** InfluxDB (Flux query mode)
- **URL:** `http://influxdb.monitoring.svc.cluster.local:8086` (prod)
- **Auth:** token-based; token sourced from Vault via ESO
- **Vault path:** `secret/terminus/monitoring/influxdb/grafana-datasource-token`

### 2. Prometheus (kube-metrics)
- **Type:** Prometheus
- **URL:** `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090`
- Assumes Prometheus is reachable within the `monitoring` namespace; if not yet deployed, datasource is registered but left as unreachable until Prometheus initiative exists.

---

## Authentication

- **Mode:** Local admin only
- Admin credentials stored in Vault: `secret/terminus/monitoring/grafana/admin`
  - Keys: `GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`
- Delivered via ESO `ExternalSecret` → k8s `Secret` → Helm `envFrom`
- No anonymous access; no OAuth in this initiative

---

## Dashboards as Code

A starter dashboard for InfluxDB metrics is provisioned via a Helm-managed `ConfigMap` with the `grafana_dashboard: "1"` label for sidecar discovery.

- **Location:** `platforms/k3s/helm/grafana/dashboards/influxdb-overview.json`
- Grafana sidecar (`grafana-sc-dashboard`) detects the ConfigMap and hot-loads it.

---

## Security

- No secrets in git.
- Admin and datasource tokens delivered via Vault + ESO.
- Grafana is internal-only (`*.trantor.internal`) — no external ingress.

---

## Ingress

- **Ingress class:** Traefik
- Manifests in `platforms/k3s/k8s/grafana-ingress/` (prod) and `platforms/k3s/k8s/grafana-dev-ingress/` (dev)
- ArgoCD apps: `grafana-ingress.yaml`, `grafana-dev-ingress.yaml`

---

## Relationship to InfluxDB Initiative

This initiative follows `terminus-infra-influxdb-monitoring` and depends on:
- `monitoring` and `monitoring-dev` namespaces already existing (created by InfluxDB initiative)
- InfluxDB service reachable at its cluster-internal DNS name
- ESO `ClusterSecretStore` active in the cluster

---

## Epics (outline)

| Epic | Title |
|------|-------|
| 1 | ArgoCD apps + Helm values (prod + dev) |
| 2 | Vault secrets + ESO wiring |
| 3 | Data source configuration |
| 4 | Starter InfluxDB dashboard provisioning |
| 5 | Verify playbooks + Semaphore templates |
