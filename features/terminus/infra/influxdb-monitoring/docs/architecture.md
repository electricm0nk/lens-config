# terminus-infra-influxdb-monitoring Architecture

This feature introduces InfluxDB as a GitOps-managed monitoring datastore for terminus infrastructure.

## Deployment

- Repo: `TargetProjects/terminus/infra/terminus.infra`
- ArgoCD apps:
  - `platforms/k3s/argocd/apps/influxdb.yaml`
  - `platforms/k3s/argocd/apps/influxdb-dev.yaml`
- Helm values:
  - `platforms/k3s/helm/influxdb/values.yaml`
  - `platforms/k3s/helm/influxdb/values-dev.yaml`

## Environments

- Prod namespace: `monitoring`
- Dev namespace: `monitoring-dev`

## Security

- No secrets in git.
- Bootstrap/admin credentials must come from Vault + ESO.

## Follow-up Requirements

- Add verify playbooks for dev/prod deployment checks.
- Add semaphore templates for verify playbooks.
- Validate Helm chart key compatibility via `helm template`.
