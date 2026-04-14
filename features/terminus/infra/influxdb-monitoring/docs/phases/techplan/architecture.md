---
stepsCompleted: [1, 2, 3, 4]
lastStep: 4
status: complete
completedAt: '2026-04-12'
workflowType: 'tech-change-architecture'
project_name: 'terminus-infra-influxdb-monitoring'
user_name: 'Todd'
date: '2026-04-12'
---

# Techplan Architecture — terminus-infra-influxdb-monitoring

## Objective

Introduce InfluxDB as a GitOps-managed monitoring datastore in terminus infrastructure with separate prod/dev applications, baseline resource controls, and clear secret-management boundaries.

## Scope

In scope:
- ArgoCD Applications for `influxdb` (prod) and `influxdb-dev`
- Helm values trees in terminus.infra for prod/dev sizing and retention defaults
- Namespace targets: `monitoring` and `monitoring-dev`
- Initial follow-up backlog for verification automation and secrets bootstrap

Out of scope:
- Grafana dashboards, alerting pipelines, Telegraf/agent rollout
- production SLO tuning and long-term retention policy finalization
- external exposure hardening beyond internal cluster access

## Technical Design

### Deployment Model

- GitOps source of truth: `TargetProjects/terminus/infra/terminus.infra`
- ArgoCD root app discovers app manifests from `platforms/k3s/argocd/apps/`
- Chart source: `https://helm.influxdata.com/`, chart `influxdb2`, target revision `2.*`
- Values source: repo-local files
  - `platforms/k3s/helm/influxdb/values.yaml`
  - `platforms/k3s/helm/influxdb/values-dev.yaml`

### Environment Separation

- Prod app: `influxdb` -> namespace `monitoring`
- Dev app: `influxdb-dev` -> namespace `monitoring-dev`
- Both apps enabled for ArgoCD auto-sync, prune, and self-heal

### Security Boundary

- No credentials/tokens in git
- InfluxDB bootstrap and admin credentials must be supplied through Vault + ESO in a follow-up story
- Verify playbooks should validate health and auth behavior without exposing secrets

## Operational Plan

1. Sync `influxdb-dev` first and verify readiness/health behavior.
2. Add dev/prod verify playbooks and semaphore templates.
3. Add Vault/ESO bootstrap path for InfluxDB admin auth material.
4. Promote to prod once dev verification is repeatable.

## Risks and Mitigations

- Chart values key drift against upstream chart:
  - Mitigation: run `helm template` validation in CI or pre-merge checks.
- Missing secret bootstrap causes failed initial auth:
  - Mitigation: enforce Vault/ESO follow-up before production rollout.
- Resource pressure in small homelab clusters:
  - Mitigation: conservative default requests/limits and smaller dev footprint.

## Exit Criteria

- Both ArgoCD apps converge to `Synced/Healthy` in cluster
- Verify playbooks exist for dev and prod
- Vault/ESO credential path is documented and operational
- `.todo` follow-up items reduced to non-blocking enhancements
