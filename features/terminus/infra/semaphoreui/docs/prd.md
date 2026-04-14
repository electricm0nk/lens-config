---
stepsCompleted:
  - step-01-init
  - step-02-discovery
  - step-02b-vision
  - step-02c-executive-summary
  - step-03-success
  - step-04-journeys
  - step-05-domain
  - step-06-innovation
  - step-07-project-type
  - step-08-scoping
  - step-09-functional
  - step-10-nonfunctional
  - step-11-polish
inputDocuments: []
workflowType: 'prd'
initiative: terminus-infra-semaphoreui
track: feature
---

# Product Requirements Document — Semaphore UI Deployment

**Author:** ToddHintzmann
**Date:** 2026-03-28
**Status:** Draft
**Initiative:** terminus-infra-semaphoreui-small

---

## Executive Summary

Deploy Semaphore UI as the first workload on the Terminus k3s platform, establishing a repeatable pattern for delivering Kubernetes workloads using the full `terminus-infra` platform stack: ArgoCD for GitOps delivery, Traefik for ingress, cert-manager with Vault PKI for TLS, ESO for secret injection from Vault, and the `postgres-databases` Tofu module for database provisioning.

Semaphore UI provides a web-based interface for running Ansible playbooks and automation jobs against the Terminus homelab. It replaces ad-hoc CLI invocations with a governed, auditable, GUI-driven execution environment — directly useful to day-to-day homelab operations.

This feature delivers Semaphore UI at `semaphore.trantor.internal` with a fully automated, GitOps-driven deployment lifecycle. It also delivers a working ArgoCD ingress at `argocd.trantor.internal`, resolving the current gap of ArgoCD having no external access point.

**Platform value:** Every pattern established here (postgres-backed workload, ESO secret injection, Traefik ingress + TLS, ArgoCD Application) becomes the reference implementation for all future `terminus-infra` and `terminus-platform` workloads.

---

## Problem Statement

1. **No automation UI**: Ansible playbooks are run ad-hoc from the CLI with no visibility, auditing, or reproducibility controls.
2. **First workload gap**: The k3s platform has been built but no workloads have been deployed through the full stack. There is no proven end-to-end delivery pattern.
3. **ArgoCD inaccessible externally**: ArgoCD is ClusterIP-only with no ingress. Access requires an active port-forward, making day-to-day use impractical.
4. **Postgres integration untested in k8s**: The `terminus-infra` postgres HA cluster has been deployed but no k8s workload has yet consumed it.

---

## Goals and Success Criteria

### Goals

- Deploy Semaphore UI as a stable, GitOps-managed workload on the Terminus k3s cluster
- Establish a fully documented, reusable pattern for postgres-backed workloads on `terminus-infra`
- Resolve ArgoCD external access by delivering a working ingress as part of this feature

### Success Criteria

| Criterion | Measure |
|---|---|
| Semaphore UI accessible | `https://semaphore.trantor.internal` loads login page with valid TLS |
| ArgoCD accessible | `https://argocd.trantor.internal` loads ArgoCD UI with valid TLS |
| Secret injection works | Admin credentials sourced from Vault via ESO — no hardcoded secrets in manifests |
| Postgres integration | Dedicated `semaphore` database provisioned via tofu module, Semaphore connects successfully |
| GitOps delivery | ArgoCD Application syncs from git; no manual `kubectl apply` in steady state |
| Persistence | `1Gi` PVC mounts correctly; Semaphore data survives pod restart |
| Certificates valid | Both ingresses use cert-manager + Vault PKI ClusterIssuer `vault-pki` |

---

## User Journeys

### Journey 1 — Run an Ansible Playbook via Semaphore UI

**Actor:** Homelab operator (CrisWeber)

1. Navigate to `https://semaphore.trantor.internal` in browser
2. Log in with admin credentials (sourced from Vault)
3. Select an existing project and task template
4. Click "Run" — provide any required variables
5. View live task output in the web interface
6. Review task history and logs after completion

**Success:** Playbook runs to completion; output visible and persisted in UI.

### Journey 2 — Bootstrap Semaphore Configuration

**Actor:** Platform operator (first-time setup)

1. Vault KV path `secret/terminus/infra/semaphore/` seeded with admin credentials
2. ESO ExternalSecret syncs to k8s Secret in `terminus-infra` namespace
3. Semaphore Deployment reads Secret as environment variables on startup
4. Admin user available at first login — no manual bootstrapping required

**Success:** Admin login works immediately after first pod start.

### Journey 3 — GitOps Deployment Update

**Actor:** Platform operator (upgrade/config change)

1. Operator updates image tag or config in git (e.g., bumps `semaphoreui/semaphore` tag)
2. ArgoCD detects drift and syncs automatically (or operator triggers manual sync)
3. Rolling update completes; new pod passes readiness probe
4. Change is traceable to a git commit

**Success:** Update applied without manual `kubectl` commands.

---

## Domain Context

- **Platform:** Terminus k3s cluster (3 CP + 5 workers, embedded etcd)
- **Namespace:** `terminus-infra`
- **Ingress controller:** Traefik, LoadBalancer VIP `10.0.0.126`, IngressClass `traefik`
- **TLS:** cert-manager ClusterIssuer `vault-pki` (Vault PKI internal CA)
- **Secrets:** ESO ClusterSecretStore `vault-backend` pulling from `vault.trantor.internal:8200` KV v2
- **Storage:** `local-path` StorageClass (default, rancher.io/local-path)
- **Database:** Patroni-managed Postgres HA cluster, primary `10.0.0.56:5432`
- **DNS:** Synology DNS, A-record `*.trantor.internal → 10.0.0.126` (or individual records per host)
- **GitOps:** ArgoCD App-of-Apps from `terminus.infra` repo `platforms/k3s/argocd/apps/`

---

## Project Classification

- **Type:** Infrastructure workload deployment (greenfield, pattern-establishing)
- **Complexity:** Medium — involves multiple platform components (postgres, ESO, Traefik, cert-manager, ArgoCD) but no custom code development
- **Risk:** Low-medium — all platform components are proven; integration sequencing is the primary risk
- **Scope:** Feature track, single sprint

---

## Functional Requirements

### FR-01: Semaphore UI Deployment

- **FR-01.1** — A Kubernetes `Deployment` runs the `semaphoreui/semaphore:latest` image in the `terminus-infra` namespace
- **FR-01.2** — The deployment mounts a `1Gi` PersistentVolumeClaim using the `local-path` StorageClass for `/tmp/semaphore` (or Semaphore's configured data directory)
- **FR-01.3** — The deployment reads admin credentials and database connection details from a Kubernetes Secret (injected by ESO)
- **FR-01.4** — A `Service` of type `ClusterIP` exposes the Semaphore pod on port `3000`
- **FR-01.5** — The deployment includes a readiness probe ensuring traffic is only routed to a healthy pod

### FR-02: Ingress and TLS

- **FR-02.1** — A Traefik `Ingress` resource routes `https://semaphore.trantor.internal` to the Semaphore Service
- **FR-02.2** — A cert-manager `Certificate` resource is issued by ClusterIssuer `vault-pki` and bound to the Semaphore Ingress
- **FR-02.3** — HTTP-to-HTTPS redirect is enforced on the Semaphore ingress

### FR-03: Secret Management

- **FR-03.1** — Vault KV path `secret/terminus/infra/semaphore/` holds all required Semaphore credentials (admin username, password, database password)
- **FR-03.2** — An ESO `ExternalSecret` in the `terminus-infra` namespace syncs the Vault path to a Kubernetes Secret `semaphore-secrets`
- **FR-03.3** — The Semaphore Deployment references `semaphore-secrets` for all credential environment variables — no plaintext credentials in any manifest

### FR-04: Database Provisioning

- **FR-04.1** — A dedicated Postgres database `semaphore` and role `semaphore` are provisioned in the Terminus Patroni cluster using the `postgres-databases` Tofu module
- **FR-04.2** — The database password for role `semaphore` is stored at Vault KV path `secret/terminus/infra/semaphore/db` and referenced by the ESO ExternalSecret
- **FR-04.3** — The Tofu state for the semaphore DB provisioning is committed to `terminus.infra` under `tofu/environments/postgres/semaphore/`

### FR-05: ArgoCD Application Registration

- **FR-05.1** — An ArgoCD `Application` manifest is added to `platforms/k3s/argocd/apps/` pointing to the Semaphore manifests in `terminus.infra`
- **FR-05.2** — The Application syncs from the `main` branch (or designated tracking ref) and deploys to the `terminus-infra` namespace
- **FR-05.3** — Sync policy is configured for automated pruning and self-healing

### FR-06: ArgoCD Ingress

- **FR-06.1** — A Traefik `Ingress` resource routes `https://argocd.trantor.internal` to the `argocd-server` Service in the `argocd` namespace
- **FR-06.2** — A cert-manager `Certificate` is issued for `argocd.trantor.internal` by ClusterIssuer `vault-pki`
- **FR-06.3** — ArgoCD server is configured to accept insecure (non-TLS) internal traffic so that Traefik handles TLS termination at the edge (i.e., `--insecure` flag or equivalent ConfigMap entry)

---

## Non-Functional Requirements

### NFR-01: Security

- **NFR-01.1** — No credentials, passwords, or tokens appear in plaintext in any file committed to git
- **NFR-01.2** — All external HTTPS endpoints use certificates issued by the internal Vault PKI CA
- **NFR-01.3** — The Semaphore Service is not exposed outside the cluster except via the Traefik Ingress

### NFR-02: Reliability

- **NFR-02.1** — Semaphore pod must restart automatically on failure (default Deployment restart policy)
- **NFR-02.2** — PVC must be bound prior to pod scheduling; deployment must not start if PVC is unbound

### NFR-03: Observability

- **NFR-03.1** — ArgoCD Application health is visible in the ArgoCD UI
- **NFR-03.2** — Standard `kubectl get/describe/logs` commands are sufficient for operational debugging — no additional monitoring tooling required for this feature

### NFR-04: Maintainability

- **NFR-04.1** — All k8s manifests are stored in the `terminus.infra` repo under a path consistent with other workloads (e.g., `platforms/k3s/manifests/semaphoreui/` or similar)
- **NFR-04.2** — The Tofu postgres DB provisioning follows the established `postgres-databases` module pattern — no custom Tofu code

---

## Scope

### In Scope

- Semaphore UI `Deployment`, `Service`, `PVC`, `Ingress`, `Certificate`, `ExternalSecret`
- Vault KV secret seeding (manual, operator-performed as prerequisite)
- ESO `ExternalSecret` manifest for `terminus-infra/semaphore-secrets`
- Tofu postgres DB provisioning for `semaphore` database and role
- ArgoCD `Application` manifest for Semaphore
- ArgoCD ingress (`argocd.trantor.internal`) with TLS

### Out of Scope

- SSO / LDAP integration for Semaphore authentication
- High-availability Semaphore (single replica only)
- Build pipeline / CI for a custom Semaphore image (upstream image used as-is)
- Monitoring/alerting integration (Prometheus, Grafana)
- Semaphore project configuration (playbooks, inventories — post-deployment operator task)

---

## Dependencies and Preconditions

| Dependency | Owner | Status |
|---|---|---|
| k3s cluster running with ArgoCD, Traefik, ESO, cert-manager | Platform | ✅ Done |
| Vault KV v2 at `secret/terminus/infra/` | Platform | ✅ Done |
| ESO ClusterSecretStore `vault-backend` functional | Platform | ✅ Done |
| Patroni Postgres cluster operational (`10.0.0.56:5432`) | Platform | ✅ Done |
| `local-path` StorageClass available | Platform | ✅ Done |
| DNS `semaphore.trantor.internal` → `10.0.0.126` | Operator | ⬜ Pending (story prerequisite) |
| DNS `argocd.trantor.internal` → `10.0.0.126` | Operator | ⬜ Pending (story prerequisite) |
| Vault KV seeded at `secret/terminus/infra/semaphore/` | Operator | ⬜ Pending (story prerequisite) |

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| ArgoCD `--insecure` flag causes redirect loop with Traefik | Medium | Medium | Use `argocd-cmd-params-cm` ConfigMap; test with curl before browser |
| cert-manager fails to issue cert (Vault PKI misconfigured) | Low | Medium | Validate ClusterIssuer `vault-pki` with a test Certificate before Semaphore cert |
| Semaphore container env var names differ from docs | Low | Low | Verify against official Semaphore environment variable reference before writing manifests |
| `local-path` PVC scheduling on wrong node after pod move | Low | Low | Document node-affinity behavior; acceptable for single-replica homelab workload |

---

## Stories (High-Level)

The following stories are expected. Final breakdown to be confirmed in DevProposal phase.

| # | Story | Dependencies |
|---|---|---|
| 1 | Provision `semaphore` postgres DB and role via Tofu | Patroni cluster healthy |
| 2 | Seed Vault KV and create ESO ExternalSecret | Vault operational |
| 3 | Write Semaphore k8s manifests (Deployment, Service, PVC) | Namespace exists |
| 4 | Write Semaphore Ingress and cert-manager Certificate | Traefik healthy, vault-pki ClusterIssuer ready |
| 5 | Add ArgoCD Application for Semaphore | Manifests in git, ArgoCD running |
| 6 | Add ArgoCD Ingress (`argocd.trantor.internal`) | Traefik healthy, vault-pki ready |

---

## Open Questions

_All open questions resolved prior to PRD completion:_

| Question | Resolution |
|---|---|
| Which Semaphore image tag? | `latest` (`semaphoreui/semaphore:latest`) |
| PVC size? | `1Gi` |
| Admin credential bootstrap approach? | ESO-injected from Vault (Option A) |
| Postgres DB provisioning in scope? | Yes — story within this feature |
| ArgoCD ingress in scope? | Yes — story within this feature |
