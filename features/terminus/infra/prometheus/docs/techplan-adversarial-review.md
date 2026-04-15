# Adversarial Review: prometheus / techplan

**Reviewed:** 2026-04-15T00:45:00Z
**Source:** phase-complete
**Overall Rating:** pass-with-warnings

---

## Summary

The `architecture.md` resolves all four deferred items from the BusinessPlan adversarial review (AR-1 through AR-4): Helm chart selection is decided (kube-prometheus-stack, ADR-001), TSDB storage is explicitly accepted as emptyDir for MVP with a documented growth path (ADR-002), the k3s kubelet TLS issue has a concrete solution (ADR-003), and dashboard delivery is specified (ADR-004). The architecture mirrors the established influxdb deployment pattern cleanly and leverages the pre-wired Prometheus datasource already in the Grafana Helm values.

Four medium-severity gaps remain. None block phase completion, but each creates silent risk at dev deployment or handoff to implementation. The party-mode challenge surfaced three additional execution concerns: dev Grafana sidecar namespace scoping, custom dashboard JSON authorship, and namespace ownership dependency confirmation. These should be reflected as open questions in the architecture frontmatter and addressed in implementation stories.

**Recommended next action:** Mark techplan complete and advance to FinalizePlan. Open three amendment issues in the implementation backlog for the medium findings.

---

## Findings

### Critical

_None._

### High

_None._

### Medium

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| M1 | Coverage Gaps | Dashboard JSON authorship — architecture specifies ConfigMap delivery but does not specify who produces the custom panel JSON for "k3s Infrastructure Health" and "Workload Health". The UX design provides panel layout and PromQL metric names, but not unit formatting, thresholds, data links, or dashboard variables. This gap will surface during the implementation story for `prometheus-infra`. | Add an implementation note or acceptance criterion in the dev story specifying that dashboard JSON must implement the UX panel spec (metric, threshold, variable) and reference Grafana community dashboard IDs 1860 (Node Exporter Full) and 3119 (Kubernetes cluster) as optional starting points. |
| M2 | Coverage Gaps | Dev Grafana sidecar namespace scoping — `grafana-dev` sidecar `searchNamespace` is not addressed. If it scans ALL namespaces, dev dashboard ConfigMaps in `monitoring-dev` will bleed into prod Grafana. If scoped to `monitoring-dev`, the architecture's silence is acceptable but unverified. | Before prod deploy, confirm grafana-dev sidecar `searchNamespace` in `values-dev.yaml`. If prod Grafana scans ALL namespaces, add `monitoring-dev` ConfigMaps to a different folder label or restrict the sidecar to `monitoring`. |
| M3 | Assumptions & Blind Spots | `forceDeployDashboards` contract unvalidated — behavior of `grafana.forceDeployDashboards: true` when `grafana.enabled: false` varies across kube-prometheus-stack minor releases. Some versions emit ConfigMaps; others silently skip them. The architecture assumes this works without a validation note. | During dev deployment, verify that `forceDeployDashboards: true` actually emits ConfigMaps in the `monitoring-dev` namespace. If it does not, fall back to embedding dashboard JSON directly in the custom ConfigMaps in `prometheus-infra`. |
| M4 | ~~Cross-Feature Dependencies~~ | **Closed** — dev environment deployment deferred; `monitoring-dev` namespace dependency is moot for this feature scope. |

### Low

| # | Dimension | Finding | Recommendation |
|---|-----------|---------|----------------|
| L1 | Complexity & Risk | emptyDir disk pressure — `--storage.tsdb.retention.time=15d` on `emptyDir` means TSDB data accumulates on the node's local storage partition (`/var/lib/rancher/k3s`). k3s nodes with small data volumes can trigger kubelet disk-pressure eviction before the 15-day window expires. No disk budget is documented. | Before prod deploy, confirm the k3s data partition has at least 2 GB free. If uncertain, add `prometheusSpec.retentionSize: 1GB` to values.yaml as a protective cap (Prometheus will roll data before disk pressure occurs). |
| L2 | Coverage Gaps | Chart version is a placeholder — `targetRevision: "67.*"` is a reasonable range but was not confirmed against the current stable release. | Before the implementation PR, check `https://github.com/prometheus-community/helm-charts/releases` and pin to the current stable minor release. Update `values.yaml` if any API changes are present. |
| L3 | Coverage Gaps | No Semaphore verify playbook — the influxdb-monitoring architecture noted "Add semaphore templates for verify playbooks" as a follow-up. That pattern is not carried forward in this architecture. The post-deployment validation checklist exists as prose only. | Deferred to Growth. Create a Semaphore task template for Prometheus target verification as a follow-on card alongside the alerting work. |

---

## Accepted Risks

| Finding | Acceptance Rationale |
|---------|----------------------|
| M2 (dev sidecar scoping) | **Closed** — dev environment deployment deferred; no monitoring-dev namespace or dev Grafana in scope. |
| M4 (monitoring-dev namespace) | **Closed** — dev environment deployment deferred. |
| L1 (emptyDir disk pressure) | **Confirmed acceptable** — current cluster disk confirmed not a concern. PVC growth path documented in ADR-002. |
| L3 (no Semaphore playbook) | Deferred to Growth alongside Alertmanager routing — consistent with established influxdb-monitoring deferred follow-up pattern. |

---

## Party-Mode Challenge

**Ekaterina (SRE / Production Ops):** The dev Grafana dashboard sidecar — where do the `monitoring-dev` ConfigMaps actually end up? If the prod sidecar scans all namespaces and `monitoring-dev` dashboards bleed into prod Grafana, we have noise in the unified pane. If the dev sidecar only watches `monitoring-dev`, we need to verify that. This is a real operational gotcha the first time someone deploys dev and sees duplicate dashboards in prod.

**Miguel (Platform Engineer):** The architecture says the `monitoring` namespace is owned by `influxdb-infra` and no namespace.yaml should be created — but this is an asserted runtime dependency, not a declared contract. If influxdb-infra is removed, the prometheus-infra resources become orphans. Also: for first deploy, CRD installation in kube-prometheus-stack and the ArgoCD sync wave aren't explicitly coordinated. If kube-prometheus-stack CRDs lose the race against anything trying to use them, you'll get `unknown resource type` failures. The sync wave model handles Helm apps, but CRDs installed by a Helm chart don't participate in wave sequencing automatically.

**Yuki (Dashboard consumer, dev team):** Who actually writes the custom dashboard JSON? The UX design has panel names and metric names, but panel JSON requires PromQL queries with range vectors, step functions, threshold colors, and variable wiring. None of that is in the architecture. The dev story is going to hit "but where does the JSON come from?" on day one. Community dashboard 1860 covers node-exporter beautifully, but the custom workload-health dashboard is a blank canvas right now.

---

## Gaps You May Not Have Considered

1. Is `grafana-dev` sidecar `searchNamespace` scoped to `monitoring-dev` or ALL namespaces? Check before deploying prometheus-dev to avoid prod dashboard bleed.
2. What is the actual disk capacity of k3s nodes — can the emptyDir TSDB accumulate 15 days of scrape data without triggering kubelet disk-pressure eviction on the k3s storage partition?
3. Who writes the PromQL for the custom dashboards, and to what spec? Is this an implementation story scope or a separate planning artifact required before the stories can be written?
4. Does `influxdb-dev-infra` actually create the `monitoring-dev` namespace, or was that namespace created manually and never codified? A first deploy would fail silently if the namespace doesn't exist.
5. kube-prometheus-stack installs ~50 CRDs — on first deploy, is there a risk of CRD installation racing with any ArgoCD health checks? Consider whether `syncOptions: ServerSideApply=true` or a CRD pre-install hook is needed.

---

## Open Questions Surfaced

These questions should be reflected back into the architecture `open_questions` frontmatter or tracked as pre-story verification tasks:

| # | Question | Owner |
|---|----------|-------|
| OQ-1 | ~~Grafana-dev sidecar~~ | **Closed** — dev environment deferred | — |
| OQ-2 | ~~k3s node disk capacity~~ | **Closed** — confirmed not a concern for current cluster | — |
| OQ-3 | Dashboard JSON authorship — community IDs or custom PromQL spec | TechPlan / dev story |
| OQ-4 | ~~monitoring-dev namespace~~ | **Closed** — dev environment deferred | — |
| OQ-5 | CRD sync wave race condition on first deploy — validate during prod deploy | Implementation |
