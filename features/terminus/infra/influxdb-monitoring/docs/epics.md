# terminus-infra-influxdb-monitoring Epics

## Epic 1: GitOps InfluxDB Deployment Foundation

- 1.1 Add prod ArgoCD app manifest (`influxdb`)
- 1.2 Add dev ArgoCD app manifest (`influxdb-dev`)
- 1.3 Add prod/dev Helm values trees
- 1.4 Validate app sync and namespace targeting

## Epic 2: Secrets and Bootstrap Hardening

- 2.1 Define Vault path conventions for InfluxDB
- 2.2 Wire ESO resources for auth material
- 2.3 Add bootstrap runbook for operators

## Epic 3: Verification and Release Integration

- 3.1 Add dev verify playbook
- 3.2 Add prod verify playbook
- 3.3 Add Semaphore templates for verify playbooks
- 3.4 Add smoke-test + triage runbook notes
