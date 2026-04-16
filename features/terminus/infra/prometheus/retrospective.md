# Retrospective: Prometheus Monitoring

**Feature:** prometheus
**Domain:** terminus / infra
**Completed:** 2026-04-16
**Track:** quickplan

---

## What Went Well

- **GitOps pattern was solid** — following the influxdb multi-source ArgoCD pattern meant Story 1-1 went smoothly with high confidence in the output
- **Dashboard ConfigMaps just work** — the Grafana sidecar ConfigMap approach (`grafana_dashboard: "1"` label + `grafana_folder` annotation) is clean and required no live cluster iteration
- **YAML + JSON embedded validation** — catching dashboard JSON issues at commit time via `yaml.safe_load` + `json.loads` prevented silent failures
- **cicd.md was prescriptive** — once consulted, the Semaphore template and verify playbook requirements were unambiguous
- **Wave ordering** — prometheus-infra (wave 1) → prometheus chart (wave 2) → prometheus-ingress (wave 3) designed correctly upfront, no rework needed

## What Didn't Go Well

- **Epic 3 was missed on first dev-complete** — the release integration epic (Traefik ingress, verify playbook, Semaphore template) was not planned during FinalizePlan and had to be added mid-close. Root cause: `cicd.md` was not consulted during planning — only during the release attempt.
- **dev-complete was premature** — phase was advanced to `dev-complete` and then reverted twice, creating noisy phase_transitions in feature.yaml
- **PR #97 included only Story 1-1 content** — Epics 2 and 3 weren't in the original PR because the session was interrupted mid-implementation; this required PR #98

## Key Learnings

1. **Consult `cicd.md` during FinalizePlan, not during dev closeout** — every new service/component needs Semaphore verify templates and a verify playbook; this should be a checklist item at planning time
2. **For infra-only components (no image tag, no DB), `ReleaseWorkflow` still requires `verify-{ServiceName}-deploy` Semaphore template** — the workflow is the same even without SeedSecrets or ProvisionDB
3. **Grafana sidecar + `grafana.enabled: false`** — `forceDeployDashboards: true` only controls built-in dashboards; custom ConfigMap dashboards work independently of this flag
4. **emptyDir + retention config** — document both together; retention without persistence is confusing to operators reading values.yaml in isolation
5. **Traefik ingress is a prerequisite for any Semaphore verify playbook** — runners on `vault.trantor.internal` cannot reach cluster-internal services

## Metrics

- **Planned duration:** 1 session (quickplan track)
- **Actual duration:** 2 sessions (second session added Epic 3)
- **Epics planned:** 2 (Epic 1: stack, Epic 2: dashboards)
- **Epics delivered:** 3 (added Epic 3: release integration)
- **Stories completed (code):** 6 of 8 (1-1, 2-1, 2-2, 3-1, 3-2, 3-3)
- **Stories deferred (cluster):** 2 (1-2, 1-3 — require live cluster post-merge)
- **PRs merged:** 1 (PR #97 — Epic 1)
- **PRs open:** 1 (PR #98 — Epics 2 + 3)
- **Adversarial review finding count:** 4 (all low/medium, none blocking)
- **Bugs found post-merge:** 0 (no cluster access in session)

## Action Items

- [ ] Merge PR #98 and complete post-merge cluster validation (Stories 1-2, 1-3)
- [ ] Run `tofu apply` for `verify_prometheus_deploy` Semaphore template, then `reconcile-semaphore.yml`
- [ ] Add "consult cicd.md for verify playbook + Semaphore template" to the FinalizePlan checklist for terminus infra services
- [ ] Future: provision PVC for Prometheus storage (replace emptyDir) — Story 4-1 backlog
- [ ] Future: add `verify-prometheus-dev-deploy.yml` when dev environment is provisioned
