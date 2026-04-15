---
feature: prometheus
doc_type: architecture
status: draft
goal: "Deploy kube-prometheus-stack to k3s via ArgoCD/Helm, closing the Kubernetes observability gap and wiring Prometheus into the existing Grafana instance for unified infra and workload visibility"
key_decisions:
  - ADR-001: kube-prometheus-stack (Prometheus Operator bundle) over standalone Prometheus
  - ADR-002: emptyDir TSDB storage for MVP; local-path PVC deferred to Growth
  - ADR-003: kubelet insecureSkipVerify=true + disable embedded k3s components (kubeProxy, kubeScheduler, kubeControllerManager, kubeEtcd)
  - ADR-004: Dashboard ConfigMaps (grafana sidecar-compatible) + forceDeployDashboards for built-in kube dashboards
open_questions:
  - Alertmanager routing targets (email/Slack/Semaphore webhook) — deferred to Growth
  - Prometheus remote-write to InfluxDB for long-term retention — deferred to Growth
  - OQ-1 Grafana-dev sidecar searchNamespace value in values-dev.yaml (verify before prometheus-dev deploy)
  - OQ-2 ~~k3s node disk capacity~~ — confirmed not a concern for current cluster; emptyDir accepted
  - OQ-3 Dashboard JSON authorship — community IDs or custom PromQL spec (resolve in dev story acceptance criteria)
  - OQ-4 monitoring-dev namespace creation source — influxdb-dev-infra or manual (verify before prometheus-dev-infra app)
  - OQ-5 CRD sync wave race condition on first deploy — validate during dev deploy
depends_on:
  - grafana (existing deployment, monitoring namespace, sidecar scanner enabled)
  - influxdb-monitoring (monitoring namespace must pre-exist)
  - k3s (cluster with node-exporter, kube-state-metrics targets)
  - secrets (Vault + ESO ClusterSecretStore; no prometheus-specific secrets for MVP)
blocks: []
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/terminus/infra/prometheus/prd.md
  - docs/terminus/infra/prometheus/ux-design.md
  - docs/terminus/infra/influxdb-monitoring/architecture.md
  - TargetProjects/terminus/infra/terminus.infra/platforms/k3s/argocd/apps/influxdb.yaml
  - TargetProjects/terminus/infra/terminus.infra/platforms/k3s/helm/grafana/values.yaml
workflowType: architecture
status: draft
updated_at: "2026-04-15T00:30:00Z"
---

# Architecture — Prometheus Monitoring

**Author:** Todd Hintzmann
**Date:** 2026-04-15
**Feature:** `prometheus` | Domain: `terminus` | Service: `infra`

---

## 1. Overview

This feature deploys `kube-prometheus-stack` to the Terminus k3s cluster, closing the Kubernetes observability gap. Prometheus scrapes Kubernetes-layer metrics; the existing Grafana instance visualizes them alongside InfluxDB (Proxmox-layer) data. No new Grafana deployment is required — the Prometheus datasource URL is already registered in the Grafana Helm values.

The deployment mirrors the established `influxdb`/`influxdb-infra` dual-app ArgoCD pattern: a wave-1 infra app (dashboard ConfigMaps) and a wave-2 Helm app (the kube-prometheus-stack chart).

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  k3s cluster — monitoring namespace                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  kube-prometheus-stack (ArgoCD app: prometheus)          │   │
│  │                                                          │   │
│  │  Prometheus Server ──────────────────────┐              │   │
│  │    emptyDir TSDB (15d retention)          │              │   │
│  │    port 9090                              │ scrapes      │   │
│  │                                           ▼              │   │
│  │  node-exporter DaemonSet (all nodes) ─ metrics          │   │
│  │  kube-state-metrics Deployment      ─ metrics           │   │
│  │  Prometheus Operator (CRD manager)                      │   │
│  │  Alertmanager (deployed, no routes)                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  prometheus-infra (ArgoCD app, wave 1)                  │    │
│  │  Dashboard ConfigMaps (grafana_dashboard: "1" label)    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌────────────────────────────────┐                            │
│  │  Grafana (existing app)        │ ← reads ConfigMaps        │
│  │  sidecar.dashboards: enabled   │   via sidecar scanner     │
│  │  datasource: Prometheus        │ ← http://prometheus-      │
│  │    (already in values.yaml)    │   kube-prometheus-        │
│  └────────────────────────────────┘   prometheus:9090         │
│                                                                 │
│  ┌────────────────────────────────┐                            │
│  │  InfluxDB (existing app)       │ ← unaffected              │
│  └────────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
                          ▲
         ArgoCD syncs from terminus.infra git repo (main)
```

**Scrape targets (in-cluster, no credentials required):**
- `node-exporter` DaemonSet — node CPU, memory, disk, network
- `kube-state-metrics` Deployment — pod phase, restart counts, resource requests/limits
- k3s kubelet endpoints (port 10250, TLS, `insecureSkipVerify: true`)
- k3s API server endpoint (cluster-internal, port 6443)

**Disabled scrape targets (k3s embeds these components; endpoints not available):**
- `kubeControllerManager` — embedded in k3s binary
- `kubeScheduler` — embedded in k3s binary
- `kubeProxy` — embedded in k3s binary
- `kubeEtcd` — embedded in k3s, not separately exposed

---

## 3. Architectural Decision Records

### ADR-001: Helm Chart — kube-prometheus-stack

**Context:** Prometheus can be deployed as a standalone server (via `prometheus-community/prometheus`) or as an Operator-managed bundle (`prometheus-community/kube-prometheus-stack`). AR-4 from the adversarial review flagged this as unresolved.

**Decision:** `kube-prometheus-stack`.

**Rationale:**
- Pre-bundles Prometheus Operator, node-exporter DaemonSet, kube-state-metrics, and Alertmanager — all of which are required by the PRD
- Built-in ServiceMonitor CRDs handle the k3s kubelet/API scrape configuration via `additionalScrapeConfigs` and `kubelet.*` value overrides
- Eliminates the need to manually define scrape configs for standard k8s targets
- Chart version pinned to `67.*` (compatible range for k3s 1.30+); lock to specific minor before production

**Trade-off accepted:** Larger chart footprint. The Grafana sub-chart (`grafana.enabled: false`) is disabled since Grafana is already deployed separately.

---

### ADR-002: TSDB Storage — emptyDir for MVP

**Context:** AR-1 from the adversarial review flagged no PV/PVC for the Prometheus TSDB. Data is lost on pod restart with `emptyDir`.

**Decision:** `emptyDir` for MVP with explicit acknowledgement of the limitation.

**Rationale:**
- The PRD success criteria state "all thresholds are advisory, not gates — success is 'it works and renders data'"
- k3s local-path provisioner requires a `PersistentVolumeClaim` spec in the chart values, which introduces drift risk if the StorageClass name varies between environments
- 15-day retention at 15s scrape interval for ~5 pods generates roughly 50–200 MB total — node-local ephemeral storage handles this without pressure
- InfluxDB already retains infrastructure time-series long-term; Prometheus serves as the real-time scrape layer

**Growth path (not in MVP):**
```yaml
# values.yaml — Growth phase amendment
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

---

### ADR-003: k3s Kubelet Endpoint — insecureSkipVerify

**Context:** AR-3 from the adversarial review: k3s kubelet binds to `127.0.0.1` on port 10255 (read-only, often disabled) and `0.0.0.0` on port 10250 (HTTPS, self-signed cert). The Prometheus Operator creates a `kubelet` Service in `kube-system` pointing to port 10250 on each node. Without configuration, scrape jobs fail due to TLS cert verification errors.

**Decision:** `kubelet.serviceMonitor.https: true` + `kubelet.serviceMonitor.insecureSkipVerify: true`.

**Rationale:**
- Avoids need to mount the cluster CA bundle into the Prometheus pod
- Standard approach for k3s deployments documented in prometheus-community issues
- Acceptable for a private cluster-internal endpoint; no external exposure

**Configuration (in values.yaml):**
```yaml
kubelet:
  serviceMonitor:
    https: true
    insecureSkipVerify: true
    cAdvisor: true
```

**Alternative considered:** Configure `bearerToken` and proper CA mount via `kubeConfigFile`. Rejected for MVP complexity.

---

### ADR-004: Dashboard Source — ConfigMaps + forceDeployDashboards

**Context:** AR-2 from the adversarial review: dashboard JSON source was unspecified.

**Decision:** Two-tier dashboard delivery:
1. **Built-in kube dashboards:** Enable `grafana.forceDeployDashboards: true` in kube-prometheus-stack values. The chart creates ConfigMaps for node-exporter, Kubernetes cluster, and workload dashboards in the `monitoring` namespace with label `grafana_dashboard: "1"`.
2. **Custom feature dashboards:** Created as raw ConfigMaps in `platforms/k3s/k8s/prometheus-infra/` — deployed by the `prometheus-infra` ArgoCD app. Two ConfigMaps: `k3s-infrastructure-health-dashboard` and `workload-health-dashboard`.

**Grafana sidecar wiring (already live):**
```yaml
# Grafana values.yaml — already deployed, no change needed
sidecar:
  dashboards:
    enabled: true
    label: grafana_dashboard
    labelValue: "1"
```

The Grafana sidecar watches the `monitoring` namespace for ConfigMaps carrying `grafana_dashboard: "1"` — both the chart-generated and custom ConfigMaps use this label.

**Dashboard folder:** Custom dashboards specify `"folder": "Terminus Infra"` in their JSON `__inputs` metadata.

---

## 4. Component Architecture

### 4.1 Prometheus Operator

The Operator is the control loop managing Prometheus and Alertmanager lifecycle. It watches for `Prometheus`, `Alertmanager`, `ServiceMonitor`, and `PrometheusRule` CRDs in the cluster.

Key configuration:
- `operator.namespaces.releaseNamespace: true` — restricts CRD watches to `monitoring`; ServiceMonitor selectors use `matchLabels`
- Operator itself runs in the `monitoring` namespace (consistent with InfluxDB pattern)

### 4.2 Prometheus Server

| Property | Value |
|----------|-------|
| Replicas | 1 (MVP; HA deferred) |
| Storage | emptyDir |
| Retention | 15d |
| Scrape interval | 15s (default) |
| Service name | `prometheus-kube-prometheus-prometheus` |
| Port | 9090 |
| Resource requests | `200m` CPU, `512Mi` memory |
| Resource limits | `500m` CPU, `1Gi` memory |

### 4.3 node-exporter (DaemonSet)

Deployed on every k3s node. Exposes host-level metrics: CPU, memory, disk, filesystem, network, hardware.

Key metrics: `node_cpu_seconds_total`, `node_memory_MemAvailable_bytes`, `node_filesystem_avail_bytes`, `node_network_transmit_bytes_total`.

Tolerations: `operator: Exists` (scrapes control-plane node too).

### 4.4 kube-state-metrics (Deployment)

Single replica. Translates k8s API objects (pods, deployments, nodes) into Prometheus metrics.

Key metrics: `kube_pod_container_status_restarts_total`, `kube_pod_container_resource_requests`, `kube_node_status_condition`.

### 4.5 Alertmanager

Deployed but without routing configuration for MVP. `alertmanager.enabled: true` — keeps the stack complete so routing config can be added without redeployment. No alert routes configured until Slack/email/webhook targets are decided (open question OQ-1).

---

## 5. Deployment Architecture

### 5.1 ArgoCD Applications

Mirrors the `influxdb` / `influxdb-infra` dual-app pattern:

| App | Wave | Source | Destination | Purpose |
|-----|------|--------|-------------|---------|
| `prometheus-infra` | 1 | `platforms/k3s/k8s/prometheus-infra` | `monitoring` | Dashboard ConfigMaps |
| `prometheus` | 2 | kube-prometheus-stack chart + values ref | `monitoring` | Prometheus stack |

> **Dev environment deferred** — dev namespace deployment skipped for this feature. Prometheus will be deployed directly to `monitoring` (prod) once validated.

> **Note:** The `monitoring` namespace is owned by the `influxdb-infra` app. Do NOT create a `namespace.yaml` in `prometheus-infra` — it would conflict. The `monitoring-dev` namespace is created by `influxdb-dev-infra`; same constraint applies.

### 5.2 Helm Values Structure

**Prod values (`platforms/k3s/helm/prometheus/values.yaml`):**
```yaml
# Disable bundled Grafana — Grafana is deployed separately
grafana:
  enabled: false
  forceDeployDashboards: true   # emit ConfigMaps for built-in kube dashboards
  sidecar:
    dashboards:
      label: grafana_dashboard
      labelValue: "1"
      namespace: monitoring

# k3s-specific: disable components embedded in k3s binary
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeProxy:
  enabled: false
kubeEtcd:
  enabled: false

# kubelet TLS: k3s uses self-signed certs on 10250
kubelet:
  serviceMonitor:
    https: true
    insecureSkipVerify: true
    cAdvisor: true

# Prometheus configuration
prometheus:
  prometheusSpec:
    retention: 15d
    scrapeInterval: 15s
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi
    # emptyDir for MVP (ADR-002)
    storageSpec: {}

alertmanager:
  enabled: true
  alertmanagerSpec:
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 100m
        memory: 128Mi
```

> **Dev values file deferred** — `values-dev.yaml` not required; dev namespace deployment is out of scope for this feature.

### 5.3 Multi-Source ArgoCD App Pattern

```yaml
# prometheus.yaml — mirrors influxdb.yaml structure
sources:
  - repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "67.*"
    helm:
      releaseName: prometheus
      valueFiles:
        - $values/platforms/k3s/helm/prometheus/values.yaml
  - repoURL: https://github.com/electricm0nk/terminus.infra.git
    targetRevision: main
    ref: values
destination:
  namespace: monitoring
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=false
```

---

## 6. Grafana Integration

### 6.1 Datasource (Already Live)

The Grafana Helm values (`platforms/k3s/helm/grafana/values.yaml`) already contain the Prometheus datasource pointing to the exact service name emitted by kube-prometheus-stack with `releaseName: prometheus`:

```yaml
# Already in Grafana values.yaml — NO CHANGE REQUIRED
- name: Prometheus
  type: prometheus
  url: http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
  access: proxy
  isDefault: false
  jsonData:
    tlsSkipVerify: true
```

This datasource will resolve automatically once the Prometheus service is running.

### 6.2 Dashboard Provisioning

The Grafana sidecar scanner is already configured:
```yaml
sidecar:
  dashboards:
    enabled: true
    label: grafana_dashboard
    labelValue: "1"
```

Dashboard ConfigMaps must:
1. Exist in the `monitoring` namespace (sidecar's default watch namespace)
2. Carry label `grafana_dashboard: "1"`
3. Have a single key in `data:` containing the dashboard JSON string

```yaml
# Example ConfigMap structure (full JSON in implementation)
apiVersion: v1
kind: ConfigMap
metadata:
  name: k3s-infrastructure-health-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  k3s-infrastructure-health.json: |
    { "title": "k3s Infrastructure Health", "uid": "k3s-infra-health", ... }
```

### 6.3 Dashboard Folder Assignment

Dashboards are assigned to folder `"Terminus Infra"` via the `__inputs` section of the dashboard JSON or via the sidecar's `folderAnnotation` mechanism. Use ConfigMap annotation:
```yaml
annotations:
  grafana_folder: "Terminus Infra"
```

---

## 7. Security Architecture

### 7.1 No Secrets for MVP

Prometheus scrapes in-cluster endpoints using the Prometheus service account's bearer token (automatically provisioned by the Operator via `ClusterRole`). No credentials required for node-exporter, kube-state-metrics, or kubelet in this topology.

### 7.2 Vault/ESO Pattern (Growth)

If remote write to InfluxDB or external alerting webhooks are added (Growth phase), credentials follow the standard pattern:
- Vault path: `secret/terminus/monitoring/prometheus/<credential>`
- ESO ExternalSecret in `platforms/k3s/k8s/prometheus-infra/external-secret.yaml`
- Injected into Prometheus via `prometheusSpec.secrets` + environment variable reference

No ESO ExternalSecret is created for the MVP deployment (consistent with no-secrets-in-git constraint).

### 7.3 RBAC

kube-prometheus-stack creates a `ClusterRole` granting Prometheus read-only access to pods, nodes, services, endpoints, and namespaces cluster-wide. This is the standard Prometheus scrape RBAC profile — no modifications needed.

For developer self-service Grafana access, the existing Grafana `viewer` role is sufficient. Prometheus datasource is read-only by default.

---

## 8. File Structure

All files are in `TargetProjects/terminus/infra/terminus.infra/`:

```
platforms/k3s/
├── argocd/apps/
│   ├── prometheus-infra.yaml          # Wave 1: dashboard ConfigMaps (prod)
│   ├── prometheus.yaml                # Wave 2: kube-prometheus-stack (prod)
│   ├── prometheus-dev-infra.yaml      # Wave 1: dashboard ConfigMaps (dev)
│   └── prometheus-dev.yaml            # Wave 2: kube-prometheus-stack (dev)
│
├── helm/prometheus/
│   └── values.yaml                    # Prod Helm values (full k3s overrides)
│
└── k8s/
    └── prometheus-infra/
        ├── k3s-infrastructure-health-dashboard.yaml   # ConfigMap: operator dashboard
        └── workload-health-dashboard.yaml             # ConfigMap: developer dashboard
```

> **No `namespace.yaml` in prometheus-infra** — `monitoring` and `monitoring-dev` namespaces are owned by `influxdb-infra` and `influxdb-dev-infra` respectively.

---

## 9. Implementation Patterns

### 9.1 GitOps Constraints

| Rule | Detail |
|------|--------|
| No `kubectl apply` in prod | All changes via PR → ArgoCD sync |
| No `helm upgrade` manually | ArgoCD owns the Helm release lifecycle |
| No secrets in git | ESO pattern for any credential (none for MVP) |
| ArgoCD sync wave discipline | `prometheus-infra` wave=1 must be Healthy before `prometheus` wave=2 |
| Dev env deferred | Dev namespace deployment skipped — deploy directly to `monitoring` (prod) |

### 9.2 Helm Values Naming Convention

- Prod values: `values.yaml` (same as influxdb pattern)
- Dev values: `values-dev.yaml` (overrides only, inherits prod base via ArgoCD `valueFiles` order)
- Do NOT fork the entire values file for dev — only override what differs

### 9.3 ArgoCD App Labeling Convention

All new apps in this namespace must carry the standard label set:
```yaml
labels:
  app.kubernetes.io/managed-by: argocd
  app.kubernetes.io/part-of: terminus-infra-k3s
  terminus.io/domain: terminus
  terminus.io/service: monitoring
  terminus.io/component: prometheus          # ← this feature's component name
  terminus.io/environment: prod | dev
```

### 9.4 ConfigMap Dashboard Key Naming

- Key in `data:` must be `<dashboard-slug>.json` (no spaces, lowercase, hyphens)
- ConfigMap `name` must be `<dashboard-slug>-dashboard`
- Folder annotation: `grafana_folder: "Terminus Infra"`

### 9.5 Validation Playbook (Post-Deployment)

After ArgoCD reports `Healthy/Synced`:
1. `kubectl -n monitoring get pods` — all pods Running
2. `kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090` → browse to `/targets` — all `up` status
3. Grafana: navigate to Connections → Data Sources → Prometheus → `Save & Test` → green
4. Grafana: dashboards folder "Terminus Infra" exists with both custom dashboards
5. Open "k3s Infrastructure Health" — node CPU panels show live data for all nodes

---

## 10. Requirements Coverage

| Requirement | Architecture Component |
|-------------|----------------------|
| FR-1.1 Prometheus deployed to `monitoring` via ArgoCD/Helm | `prometheus` ArgoCD app, `kube-prometheus-stack` chart |
| FR-1.2 Dev deployment to `monitoring-dev` | `prometheus-dev` ArgoCD app, `values-dev.yaml` |
| FR-1.3 Scrape node-exporter all nodes | `node-exporter` DaemonSet (bundled), toleration `Exists` |
| FR-1.4 Scrape kube-state-metrics | `kube-state-metrics` Deployment (bundled) |
| FR-1.5 Scrape k3s control-plane metrics | kubelet ServiceMonitor, `insecureSkipVerify: true` (ADR-003) |
| FR-2.1 Grafana datasource provisioned | Already in `grafana/values.yaml` — live on Prometheus deploy |
| FR-2.2 datasource `isDefault: false` | Explicit in existing Grafana values |
| FR-2.3 "Save & Test" passes | ADR-003 resolves TLS; service name matches pre-wired URL |
| FR-3.1 Operator dashboard deployed | `grafana.forceDeployDashboards: true` (ADR-004) |
| FR-3.2 "k3s Infrastructure Health" dashboard rendered | Custom ConfigMap in `prometheus-infra` k8s manifests |
| FR-3.3 "Workload Health" dashboard rendered | Custom ConfigMap in `prometheus-infra` k8s manifests |
| FR-3.4 `$namespace` filter on Workload Health | PromQL variable `label_values(kube_pod_info, namespace)` in dashboard JSON |
| FR-4.1 15d retention | `prometheusSpec.retention: 15d` |
| FR-4.2 15s scrape interval | `prometheusSpec.scrapeInterval: 15s` |
| FR-4.3 Resource limits set | `resources.limits` in `prometheusSpec` |
| FR-5.1 No credentials in git | No ESO ExternalSecret for MVP (no credentials required) |
| FR-5.2 Alertmanager deployed (unconfigured) | `alertmanager.enabled: true`, no routes |
| NFR-SEC-1 No secrets in git | Affirmed — entire scrape target set requires no credentials |
| NFR-REL-1 Self-healing sync | `selfHeal: true` in syncPolicy |
| NFR-MAINT-1 GitOps-only changes | ArgoCD multi-source pattern enforces this |
