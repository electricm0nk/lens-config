---
phase: techplan
initiative: terminus-infra-semaphoreui
date: 2026-03-28
status: approved
---

# Architecture — Semaphore UI Deployment

**Initiative:** terminus-infra-semaphoreui  
**Track:** feature  
**Author:** ToddHintzmann  
**Date:** 2026-03-28  
**Status:** Approved for implementation planning

---

## Overview

Deploy Semaphore UI as the first concrete workload on the Terminus k3s platform. The primary goal is not just to deliver the Semaphore automation interface, but to establish and validate the full end-to-end delivery pattern for postgres-backed, secret-managed, GitOps-deployed workloads on `terminus-infra`. Every decision documented here becomes the reference implementation for future workloads.

This architecture covers:
1. Kubernetes manifest structure and resource design for Semaphore UI
2. Persistent volume strategy
3. Secret management via ESO + Vault
4. Database provisioning via OpenTofu
5. Ingress, TLS, and DNS
6. ArgoCD GitOps delivery
7. ArgoCD ingress (resolving the ArgoCD external access gap)
8. Deployment sequencing

---

## Platform Context

| Component | Implementation | Detail |
|---|---|---|
| Cluster | k3s | 3 control-plane + 5 workers, embedded etcd |
| Namespace | `terminus-infra` | Pre-existing, all infra workloads |
| Ingress controller | Traefik | LoadBalancer VIP `10.0.0.126`, IngressClass `traefik` |
| TLS | cert-manager | ClusterIssuer `vault-pki` (Vault internal PKI CA) |
| Secrets | ESO | ClusterSecretStore `vault-backend` → Vault KV v2 at `vault.trantor.internal:8200` |
| Storage | `local-path` | Rancher `rancher.io/local-path`, ReadWriteOnce |
| Database | Patroni Postgres HA | Primary at `10.0.0.56:5432` |
| GitOps | ArgoCD | App-of-Apps from `terminus.infra` repo `platforms/k3s/argocd/apps/` |
| DNS | Synology | `*.trantor.internal → 10.0.0.126` (or individual records) |

---

## Component Design

### 1. Repository Layout (`terminus.infra`)

All Kubernetes manifests are authored in the `terminus.infra` git repository. Semaphore UI manifests follow a per-resource split (one file per Kubernetes resource kind) for maximum readability, reviewability, and diff clarity.

```
terminus.infra/
├── platforms/k3s/
│   ├── manifests/
│   │   ├── terminus-infra/
│   │   │   └── semaphoreui/
│   │   │       ├── deployment.yaml
│   │   │       ├── service.yaml
│   │   │       ├── pvc.yaml
│   │   │       ├── externalsecret.yaml
│   │   │       ├── certificate.yaml
│   │   │       └── ingress.yaml
│   │   └── argocd/
│   │       ├── certificate.yaml
│   │       └── ingress.yaml
│   └── argocd/
│       └── apps/
│           └── semaphoreui.yaml
└── tofu/
    └── environments/
        └── postgres/
            └── semaphore/
                ├── main.tf
                ├── variables.tf
                └── backend.tf
```

---

### 2. Kubernetes Manifests

#### 2.1 Namespace

Semaphore UI deploys into the pre-existing `terminus-infra` namespace. No new namespace is created.

#### 2.2 PersistentVolumeClaim

```yaml
name: semaphoreui-data
storageClass: local-path
accessModes: [ReadWriteOnce]
capacity: 1Gi
```

`local-path` is a Rancher-managed hostPath provisioner. The PVC binds to the node where the pod first schedules and follows that node for the lifetime of the PVC. This is acceptable for a single-replica homelab workload. If the pod is evicted to a different node, the PVC will not follow — the pod must be pinned or the PVC re-created to move nodes.

#### 2.3 Deployment

```yaml
namespace: terminus-infra
replicas: 1   # constrained to 1 — local-path PVC is ReadWriteOnce
image: semaphoreui/semaphore:latest
imagePullPolicy: Always   # required when tracking 'latest' to ensure current image on restart
volumeMounts:
  - name: data
    mountPath: /tmp/semaphore   # Semaphore default data directory
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    memory: 512Mi
    # No CPU limit — avoids throttling for interactive web UI
```

The `replicas: 1` constraint is driven by the `ReadWriteOnce` access mode of the `local-path` PVC. Multi-replica Semaphore with SQLite is also unsupported upstream.

Readiness probe: HTTP GET on port `3000` (Semaphore default) at `/api/ping` or root — standard HTTP readiness check to gate traffic routing.

#### 2.4 Service

```yaml
name: semaphoreui
namespace: terminus-infra
type: ClusterIP
port: 3000   # Semaphore default HTTP port
targetPort: 3000
```

ClusterIP only — no external exposure except via the Traefik Ingress.

#### 2.5 ExternalSecret

```yaml
name: semaphoreui-externalsecret
namespace: terminus-infra
secretStoreRef:
  name: vault-backend
  kind: ClusterSecretStore
vaultPath: secret/terminus/infra/semaphoreui
targetSecret: semaphoreui-secrets
fields:
  admin_user        → SEMAPHORE_ADMIN
  admin_password    → SEMAPHORE_ADMIN_PASSWORD
  admin_email       → SEMAPHORE_ADMIN_EMAIL
  db_password       → SEMAPHORE_DB_PASS
  access_key_encryption → SEMAPHORE_ACCESS_KEY_ENCRYPTION
```

All five fields are co-located in a single Vault KV path. A single `ExternalSecret` syncs them all to one k8s Secret (`semaphoreui-secrets`). The db password is not stored at a separate path — keeping it co-located avoids a second `ExternalSecret` and Vault policy.

The k8s Secret `semaphoreui-secrets` is consumed by the Deployment as `envFrom.secretRef`.

**Vault KV prerequisites (operator manual action before deployment):**
```
vault kv put secret/terminus/infra/semaphoreui \
  admin_user=<admin-username> \
  admin_password=<strong-password> \
  admin_email=<admin-email> \
  db_password=<postgres-role-password> \
  access_key_encryption=<32+-char-random-string>
```

#### 2.6 Database Environment Variables

Postgres connection details are split between plain manifest env vars (non-secret) and the k8s Secret (password only):

```yaml
# In Deployment manifest (plaintext — not secret):
env:
  - name: SEMAPHORE_DB_DIALECT
    value: postgres
  - name: SEMAPHORE_DB_HOST
    value: "10.0.0.56"
  - name: SEMAPHORE_DB_PORT
    value: "5432"
  - name: SEMAPHORE_DB_NAME
    value: semaphore
  - name: SEMAPHORE_DB_USER
    value: semaphore

# From Secret (via envFrom or explicit env valueFrom):
  - name: SEMAPHORE_DB_PASS
    valueFrom:
      secretKeyRef:
        name: semaphoreui-secrets
        key: SEMAPHORE_DB_PASS
```

#### 2.7 Certificate

An explicit `Certificate` resource is created rather than relying on cert-manager ingress annotations. This makes cert status independently inspectable (`kubectl get certificate -n terminus-infra`) and decouples certificate lifecycle from ingress changes.

```yaml
name: semaphoreui-tls-cert
namespace: terminus-infra
issuerRef:
  kind: ClusterIssuer
  name: vault-pki
dnsNames: [semaphore.trantor.internal]
secretName: semaphoreui-tls
```

#### 2.8 Ingress

```yaml
name: semaphoreui
namespace: terminus-infra
ingressClassName: traefik
tls:
  secretName: semaphoreui-tls   # populated by cert-manager Certificate above
rules:
  - host: semaphore.trantor.internal
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: semaphoreui
              port: 3000
```

HTTP→HTTPS redirect is handled at the Traefik IngressClass level (platform default). No per-ingress annotation required.

---

### 3. PostgreSQL Database Provisioning (OpenTofu)

A standalone OpenTofu environment provisions the `semaphore` database and role in the Patroni Postgres HA cluster.

**Environment path:** `tofu/environments/postgres/semaphore/`

This is a standalone environment directory (not merged with any existing `postgres/dev/` environment). The `postgres/dev/` environment manages the Postgres cluster provisioning itself; `postgres/semaphore/` manages database-level resources within it.

**Module used:** The existing `postgres-databases` module (already established in the repo). No custom Tofu code is required.

**Provisioned resources:**
- Database: `semaphore`
- Role: `semaphore` (with password sourced from Vault — passed as variable, not hardcoded)
- Permissions: `GRANT ALL ON DATABASE semaphore TO semaphore`

**Tofu state backend:** Consul (per established pattern), key `tofu/postgres/semaphore`.

The database role password must match what is stored in Vault at `secret/terminus/infra/semaphoreui.db_password`. The operator seeds Vault first, then passes the value to Tofu via `-var` or a `tfvars` file (not committed to git).

---

### 4. ArgoCD Application

An ArgoCD `Application` resource registers Semaphore UI for GitOps delivery.

**Manifest path:** `platforms/k3s/argocd/apps/semaphoreui.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: semaphoreui
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <terminus.infra-repo-url>
    targetRevision: main
    path: platforms/k3s/manifests/terminus-infra/semaphoreui
  destination:
    server: https://kubernetes.default.svc
    namespace: terminus-infra
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

**Sync policy:** Fully automated with self-heal and prune. This is the reference GitOps pattern. Changes committed to `main` in `terminus.infra` are automatically applied to the cluster. No manual `kubectl apply` in steady state.

**Branch tracking:** `main` (HEAD). Tag/SHA pinning is deferred — not required for homelab operations.

---

### 5. ArgoCD Ingress

The ArgoCD UI is currently ClusterIP-only with no external access. This feature resolves that gap as a co-delivered sub-feature.

#### 5.1 ArgoCD Insecure Mode

ArgoCD's server runs with internal TLS by default. To allow Traefik to terminate TLS at the edge (single-hop TLS), ArgoCD must be configured to accept plain HTTP internally:

```yaml
# platforms/k3s/manifests/argocd/configmap-patch.yaml (or inline patch)
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"
```

This avoids modifying the ArgoCD Deployment spec directly and survives ArgoCD upgrades.

> **Note:** After applying this ConfigMap, the `argocd-server` Deployment must be restarted for the change to take effect (`kubectl rollout restart deployment argocd-server -n argocd`).

#### 5.2 ArgoCD Certificate

```yaml
name: argocd-tls-cert
namespace: argocd
issuerRef:
  kind: ClusterIssuer
  name: vault-pki
dnsNames: [argocd.trantor.internal]
secretName: argocd-tls
```

#### 5.3 ArgoCD Ingress

**Manifest path:** `platforms/k3s/manifests/argocd/ingress.yaml`

```yaml
name: argocd
namespace: argocd
ingressClassName: traefik
tls:
  secretName: argocd-tls
rules:
  - host: argocd.trantor.internal
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: argocd-server
              port: 80   # plain HTTP from Traefik to ArgoCD server (insecure mode)
```

---

### 6. DNS

Both ingress hostnames require DNS A-records pointing to the Traefik LoadBalancer VIP:

| Hostname | IP | Provider |
|---|---|---|
| `semaphore.trantor.internal` | `10.0.0.126` | Synology DNS |
| `argocd.trantor.internal` | `10.0.0.126` | Synology DNS |

These are operator manual actions (or may already exist if a wildcard `*.trantor.internal` A-record is in place). DNS must be confirmed before testing ingress.

---

## Deployment Sequencing

The following order is required due to dependencies between components:

| Step | Action | Mechanism | Owner |
|---|---|---|---|
| 1 | Seed Vault KV at `secret/terminus/infra/semaphoreui` | `vault kv put` (manual) | Operator |
| 2 | Provision `semaphore` Postgres DB and role | `tofu apply` in `postgres/semaphore/` | Operator |
| 3 | Apply ESO `ExternalSecret` | ArgoCD sync (or `kubectl apply`) | Operator / ArgoCD |
| 4 | Apply Semaphore `Deployment`, `Service`, `PVC` | ArgoCD sync | ArgoCD |
| 5 | Apply Semaphore `Certificate` and `Ingress` | ArgoCD sync | ArgoCD |
| 6 | Apply ArgoCD `argocd-cmd-params-cm` patch (insecure mode) | `kubectl apply` / ArgoCD | Operator |
| 7 | Restart `argocd-server` deployment | `kubectl rollout restart` | Operator |
| 8 | Apply ArgoCD `Certificate` and `Ingress` | `kubectl apply` (not via ArgoCD App-of-Apps self-reference) | Operator |
| 9 | Add DNS records for both hostnames | Synology DNS admin UI | Operator |
| 10 | Verify `semaphore.trantor.internal` and `argocd.trantor.internal` load in browser | Browser / curl | Operator |

> **Note on Step 8:** The ArgoCD ingress manifests cannot be delivered by ArgoCD itself (bootstrapping problem — ArgoCD must be accessible to apply its own ingress). Apply these manifests via direct `kubectl apply` once ArgoCD insecure mode is confirmed.

---

## Security Model

- No credentials, passwords, or tokens appear in plaintext in any committed manifest
- All secrets flow through Vault → ESO → k8s Secret
- Semaphore Service is ClusterIP only — not reachable outside the cluster except through Traefik ingress
- All external HTTPS endpoints use certificates issued by the internal Vault PKI CA (`vault-pki`)
- `SEMAPHORE_ACCESS_KEY_ENCRYPTION` is a 32+ character random string stored in Vault — used by Semaphore to encrypt stored access keys at rest

---

## Operational Notes

- **Single replica:** Semaphore does not support HA with `local-path` PVC. This is acceptable for homelab operations.
- **`imagePullPolicy: Always`:** Required for `latest` tag. Ensures the current upstream image is used on every pod restart.
- **PVC node binding:** `local-path` binds to the scheduling node. If the pod moves to another node, the PVC will not follow. For production use, a distributed StorageClass would be required.
- **ArgoCD insecure mode:** Does not affect encryption in transit — Traefik handles TLS termination at the edge. Traffic between Traefik and ArgoCD is cluster-internal.
- **Cert-manager integration:** Semaphore certificate is provisioned by cert-manager using the `vault-pki` ClusterIssuer. If the ClusterIssuer is not responding, certificate issuance will stall. Verify `vault-pki` ClusterIssuer status before deploying.

---

## Implementation Readiness

| Check | Status | Notes |
|---|---|---|
| All platform components available | ✅ | ArgoCD, Traefik, ESO, cert-manager, Patroni all running |
| Manifest structure defined | ✅ | Per-resource split, paths specified |
| Secret strategy defined | ✅ | Single Vault path, one ExternalSecret, five fields |
| DB provisioning path defined | ✅ | Standalone Tofu env, `postgres-databases` module |
| ArgoCD sync policy defined | ✅ | Automated, self-heal, prune |
| ArgoCD insecure workaround defined | ✅ | `argocd-cmd-params-cm` ConfigMap |
| Deployment sequence defined | ✅ | 10-step ordered sequence documented |
| DNS prerequisites identified | ✅ | Two records, operator action |
| Security: no plaintext secrets | ✅ | All credentials through Vault → ESO |
| Stories derivable | ✅ | 6 stories correspond to PRD high-level breakdown |
