---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
inputDocuments:
  - docs/terminus/infra/k3s/product-brief.md (from terminus-infra-k3s-small-preplan)
  - docs/terminus/infra/k3s/brainstorm-notes.md (from terminus-infra-k3s-small-preplan)
workflowType: architecture
lastStep: 8
status: complete
project_name: terminus-infra-k3s
user_name: Todd
date: '2026-03-28'
completedAt: '2026-03-28'
---

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**

- k3s cluster bootstrap and node join process (control-plane + worker topology)
- Namespace model for service separation across infra and platform workloads
- Ingress baseline for downstream service exposure
- Storage class and PV baseline for stateful workloads
- RBAC and service account foundations at the cluster level
- Cert-manager for in-cluster TLS certificate lifecycle (decided: in substrate scope)
- Cluster upgrade workflow
- Node replacement workflow
- Bootstrap and recovery documentation
- Downstream consumer contract — explicit handoff to `terminus-infra-secrets`, `terminus-infra-postgres`, `terminus-platform-temporal`

**Non-Functional Requirements:**

- **Repeatability** — cluster bootstrap 100% reproducible from committed artifacts
- **Security** — no secrets in bootstrap manifests or cluster code; explicit access boundaries
- **Recoverability** — node replacement and cluster rebuild documented and validated from the start
- **Consumer contract stability** — downstream features have deterministic assumptions about namespaces, ingress, storage, and TLS
- **Bootstrap ordering guarantee** — "cluster ready" is a defined, testable state; not prose-only

**Scale & Complexity:**

- Primary domain: infrastructure / operations
- Complexity: medium — no application logic; significant operational lifecycle surface
- Architectural components: cluster topology, bootstrap, ingress, storage baseline, namespaces/RBAC, cert-manager, operational lifecycle

### External Dependencies (Pre-conditions)

| Dependency | Location | Contract |
|------------|----------|----------|
| Vault (secrets) | External VM — `vault.trantor.internal` | Must be reachable before cluster bootstrap. Not cluster-owned. |
| Proxmox VMs | `terminus-infra-proxmox` | VMs provisioned before cluster bootstrap begins. |
| DNS resolver | Synology NAS — `trantor.internal` domain | Cluster nodes and workloads resolve hostnames via this resolver. |
| Prometheus | External VM (location TBD) | Out of scope for this feature. Cluster exposes scrape endpoints only. |

**Bootstrap ordering:** Vault and DNS must be reachable before cluster bootstrap. The k3s feature does not provision them — their availability is a documented pre-condition.

### Scope Boundary — Hard Stop Line

> Anything with its own lifecycle, versioning, or configuration surface area specific to a consuming service stays downstream. The substrate is done when: nodes are healthy, namespaces exist, ingress controller is present, storage class is available, cert-manager is running, and the consumer contract is documented.

**In scope (cluster substrate):**
- k3s install, node join, control-plane topology
- Ingress controller, namespace model, RBAC baseline
- Storage class definition (Longhorn vs local-path — deferred ADR, criteria captured below)
- Cert-manager (in-cluster TLS lifecycle)
- Operational runbooks (bootstrap, upgrade, node replacement)
- Consumer contract documentation

**Out of scope:**
- Vault deployment and PKI configuration → `terminus-infra-secrets`
- Prometheus deployment and alerting → separate feature (TBD)
- Postgres → `terminus-infra-postgres`
- Temporal → `terminus-platform-temporal`
- Application-specific Helm releases or workload deployments

### Cross-Cutting Concerns

- **Bootstrap ordering** — Vault and DNS are pre-conditions; architecture must document the dependency chain explicitly
- **TLS everywhere** — cert-manager in the substrate means every downstream feature can assume TLS is available without owning it
- **Storage decision impact** — storage class choice propagates to all stateful workloads; evaluation criteria captured as an ADR
- **Metrics exposure** — cluster exposes node-exporter, kube-state-metrics, kubelet endpoints; external Prometheus scrapes them
- **Secret handling** — bootstrap process must not commit credentials; Vault-backed secret injection pattern applies from day one

### Deferred Architectural Decision Record

**ADR-001: Storage Class Selection**
- **Status:** Deferred — decide at first stateful workload with real requirements
- **Default:** k3s local-path-provisioner (node-local, non-replicated)
- **Upgrade candidates:** Longhorn (in-cluster block storage, replication), NFS-backed (external dependency), TopoLVM (LVM-based, higher ops complexity)
- **Decision criteria:** node-failure tolerance requirement, replication needs, ops complexity tolerance, whether microservice DBs land on k3s
- **Trigger:** first stateful workload that cannot accept node-local PV loss

---

## Toolchain & Foundation Decisions (Step 3)

### Decided

| Decision | Choice | Rationale |
|----------|--------|-----------|
| k3s install method | Ansible shell / install script (Option B) | Consistent with existing ansible-core 2.19 toolchain; no extra binary dependency |
| In-cluster component delivery | Helm | Standard package management for third-party components; upgrade story, values override pattern |
| Ingress controller | Traefik (k3s default) | Bundled with k3s; reduces install surface; adequate for homelab/proxmox substrate |
| LoadBalancer | MetalLB (L2 mode) | Standard bare-metal LB; L2 mode appropriate for non-BGP homelab network |
| GitOps layer | ArgoCD | Familiar from professional context; in-scope for substrate (cluster-level app delivery) |

### ArgoCD Scope Note

ArgoCD is in the substrate — it is the **delivery mechanism** for cluster components, not a downstream feature. All Helm releases (cert-manager, MetalLB, Traefik configuration, namespace scaffolding) are delivered via ArgoCD `Application` resources. Downstream features (`terminus-infra-secrets`, `terminus-infra-postgres`, `terminus-platform-*`) will register their own ArgoCD apps pointing at their own repos.

---

## Core Architectural Decisions (Step 4)

### Security & Access

| Decision | Choice | Detail |
|----------|--------|--------|
| Cluster API access | Direct kubeconfig | Internal IP; operator must be on-network or VPN. No LB VIP for API server. |
| Vault integration | External Secrets Operator (ESO) | In-substrate cluster operator; syncs Vault KV → k8s Secrets. All downstream features assume ESO is available. |

### Infrastructure & Delivery

| Decision | Choice | Detail |
|----------|--------|--------|
| ArgoCD bootstrap | Ansible-first | Ansible performs one-time `helm install argocd`; ArgoCD then manages itself and all cluster components via App-of-Apps pattern. |
| GitOps repo | `terminus.infra` (this repo) | ArgoCD `Application` resources committed alongside Ansible playbooks. No separate gitops repo for MVP. |

### Cluster Topology

| Decision | Choice | Detail |
|----------|--------|--------|
| Control-plane count | 3-node HA | etcd quorum; survives 1 node failure. All 3 nodes provisioned via Proxmox + Ansible. |
| Control-plane VM size | 2 vCPU, shared | Shared (time-sliced) Proxmox vCPUs. Adequate for homelab scale. etcd latency sensitivity addressed by low host contention, not CPU pinning. |
| Worker node count | 5 workers | Room to grow. Workloads distributed across all 5; add nodes without cluster rebuild. |
| Node scheduling model | Role-based taints | Control-plane nodes: `node-role.kubernetes.io/control-plane:NoSchedule`. Workloads land on dedicated worker nodes only. |

### Consumer Contract (Downstream Assumptions)

Any downstream feature that declares a dependency on `terminus-infra-k3s` can assume:

1. k3s cluster is healthy — all nodes `Ready`, system namespaces up
2. Namespaces exist per the namespace model (defined in Step 6)
3. ESO is installed and Vault connectivity is established
4. cert-manager is running with a working `ClusterIssuer`
5. MetalLB is configured with an IP pool for `LoadBalancer` services
6. Traefik is running and `IngressClass: traefik` is available
7. ArgoCD is running and the App-of-Apps root app is synced
8. Default storage class is set (local-path provisioner unless overridden by ADR-001)

---

## Implementation Patterns & Consistency Rules (Step 5)

### Kubernetes Resource Naming

| Resource type | Pattern | Example |
|---|---|---|
| Namespaces | `{domain}-{service}` | `terminus-infra`, `terminus-platform` |
| ArgoCD Applications | `{domain}-{service}-{component}` | `terminus-infra-cert-manager` |
| Helm releases | match ArgoCD app name | `terminus-infra-cert-manager` |
| k8s Secrets (ESO-managed) | `{service}-{secret-name}` | `infra-vault-token` |
| RBAC: ClusterRoles | `{domain}:{role}` | `terminus:infra-admin` |
| RBAC: ServiceAccounts | `{service}-sa` | `argocd-sa` |

### Standard Labels

Every managed resource carries these labels — no exceptions:

```yaml
labels:
  app.kubernetes.io/managed-by: argocd       # or ansible for bootstrap-only resources
  app.kubernetes.io/part-of: terminus-infra  # domain-service of the owning feature
  terminus.io/domain: terminus
  terminus.io/service: infra                 # or platform, secrets, postgres, etc.
```

### Helm Values Structure

One `values.yaml` per component, committed in this repo. No inline `--set` in Ansible or ArgoCD spec.

```
infra/k3s/helm/
  cert-manager/values.yaml
  metallb/values.yaml
  traefik/values.yaml
  argocd/values.yaml
  external-secrets/values.yaml
```

### ArgoCD App-of-Apps Pattern

```
infra/k3s/argocd/
  root-app.yaml          # Installed by Ansible; points to infra/k3s/argocd/apps/
  apps/
    cert-manager.yaml
    metallb.yaml
    traefik.yaml
    external-secrets.yaml
    namespaces.yaml
```

All `Application` resources: `syncPolicy.automated.prune: true`, `selfHeal: true`.

### Ansible Conventions

- One playbook per lifecycle phase: `bootstrap.yml`, `upgrade.yml`, `node-replace.yml`
- No hardcoded IPs — all hosts via inventory group names
- Vault secrets via `community.hashi_vault.hashi_vault_kv2_get` only
- All tasks have `name:` — no anonymous tasks

### Directory Conventions

- All cluster state files under `infra/k3s/`
- Ansible under `infra/k3s/ansible/`
- Docs under `docs/terminus/infra/k3s/`
- No mixing of Ansible and Helm files in the same directory

---

## Project Structure & Boundaries (Step 6)

### Complete Directory Tree

```
termus.infra/
├── README.md
├── .gitignore
├── .sops.yaml                          # sops encryption config (age keys)
│
├── infra/
│   └── k3s/
│       ├── ansible/
│       │   ├── inventory/
│       │   │   ├── hosts.ini           # Node groups: control_plane, workers
│       │   │   └── group_vars/
│       │   │       ├── all.yml         # Common vars (cluster name, domain, etc.)
│       │   │       ├── control_plane.yml
│       │   │       └── workers.yml
│       │   ├── roles/
│       │   │   ├── k3s-server/         # Install k3s + config on control-plane nodes
│       │   │   ├── k3s-agent/          # Join worker nodes to cluster
│       │   │   └── argocd-bootstrap/   # One-time: helm install argocd
│       │   ├── bootstrap.yml           # Full cluster bootstrap playbook
│       │   ├── upgrade.yml             # Cluster upgrade playbook
│       │   └── node-replace.yml        # Node drain/replace/rejoin playbook
│       │
│       ├── helm/
│       │   ├── argocd/values.yaml
│       │   ├── cert-manager/values.yaml
│       │   ├── external-secrets/values.yaml
│       │   ├── metallb/values.yaml
│       │   └── traefik/values.yaml
│       │
│       ├── argocd/
│       │   ├── root-app.yaml           # Installed by Ansible; self-manages apps/
│       │   └── apps/
│       │       ├── cert-manager.yaml
│       │       ├── external-secrets.yaml
│       │       ├── metallb.yaml
│       │       ├── namespaces.yaml
│       │       └── traefik.yaml
│       │
│       └── manifests/
│           ├── namespaces/
│           │   └── namespaces.yaml     # All terminus-* namespaces
│           ├── rbac/
│           │   ├── cluster-roles.yaml
│           │   └── service-accounts.yaml
│           ├── metallb/
│           │   └── ip-pool.yaml        # MetalLB L2 IP pool
│           └── cert-manager/
│               └── cluster-issuer.yaml # ClusterIssuer (Vault PKI or ACME)
│
└── docs/
    └── terminus/
        └── infra/
            └── k3s/
                ├── architecture.md
                ├── runbook-bootstrap.md
                ├── runbook-upgrade.md
                └── runbook-node-replace.md
```

### Architectural Boundaries

| Boundary | Owner | What crosses it |
|---|---|---|
| `infra/k3s/ansible/` → cluster | Ansible | Installs k3s, bootstraps ArgoCD; then hands off permanently |
| `infra/k3s/argocd/` → cluster | ArgoCD | All Helm releases and manifests after bootstrap |
| cluster → `vault.trantor.internal` | ESO | Secret sync at pod startup; read-only from cluster |
| cluster → external Prometheus | node-exporter, kubelet | Scrape endpoints exposed; no push from cluster |
| `terminus-infra-k3s` → downstream | Consumer contract | Namespaces, ESO, cert-manager, MetalLB, Traefik, ArgoCD all assumed present |

---

## Architecture Validation (Step 7)

### Coherence Validation ✅

| Check | Result |
|---|---|
| Ansible + k3s install script | ✅ Standard orchestration layer |
| ArgoCD + Helm values files | ✅ `helm.valueFiles` in Application spec references committed values |
| ESO + Vault external | ✅ ESO supports Vault KV v2 via `SecretStore` CRD natively |
| MetalLB L2 + Traefik LoadBalancer | ✅ MetalLB provides VIP; Traefik `type: LoadBalancer` works directly |
| cert-manager + Traefik TLS | ✅ cert-manager issues certs; Traefik references via `Certificate` + `secretName` |
| 3-node HA + k3s embedded etcd | ✅ k3s embedded etcd supports HA; no external etcd required |
| Role-based taints + ArgoCD/ESO scheduling | ✅ Resolved — `tolerations` must be absent for `NoSchedule` in `helm/argocd/values.yaml` and `helm/external-secrets/values.yaml` |
| sops+age + Ansible Vault secrets | ✅ sops encrypts at rest; Ansible uses `community.hashi_vault` for runtime — no conflict |

### Requirements Coverage ✅

| Functional Requirement | Architectural Support |
|---|---|
| k3s bootstrap and node join | ✅ Ansible roles: `k3s-server`, `k3s-agent`, `bootstrap.yml` |
| Namespace model | ✅ `manifests/namespaces/namespaces.yaml` — ArgoCD-managed |
| Ingress baseline | ✅ Traefik (bundled), `helm/traefik/values.yaml`, ArgoCD app |
| Storage class baseline | ✅ local-path default; ADR-001 deferred with criteria |
| RBAC baseline | ✅ `manifests/rbac/` |
| cert-manager | ✅ Helm + ArgoCD + `manifests/cert-manager/cluster-issuer.yaml` |
| Cluster upgrade | ✅ `ansible/upgrade.yml` |
| Node replacement | ✅ `ansible/node-replace.yml` |
| Bootstrap + recovery docs | ✅ `docs/terminus/infra/k3s/runbook-*.md` |
| Downstream consumer contract | ✅ Explicit contract table in Step 4 |

| NFR | Status |
|---|---|
| Repeatability | ✅ 100% from committed artifacts |
| Security (no secrets in repo) | ✅ sops+age at rest; Vault via ESO at runtime |
| Recoverability | ✅ Runbooks committed alongside cluster code |
| Bootstrap ordering guarantee | ✅ Pre-conditions documented; Ansible enforces sequence |
| Consumer contract stability | ✅ Explicit contract in Step 4 |

### Resolved Gaps

**Gap 1 — ClusterIssuer type:** Vault PKI (internal CA). `cluster-issuer.yaml` will configure a `VaultIssuer` or `ClusterIssuer` pointing at `vault.trantor.internal`. All issued certs are signed by the internal Vault CA. External-facing TLS (if needed later) is a separate ADR.

**Gap 2 — MetalLB IP pool:** `10.0.0.126 – 10.0.0.254`. Reserved LAN range for k3s `LoadBalancer` services. `manifests/metallb/ip-pool.yaml` will configure an `IPAddressPool` and `L2Advertisement` for this range. **Confirmed clear of DHCP scope.**

**Gap 3 — Namespace list:** Resolved — see Namespace Model below.

**Gap 4 — ArgoCD toleration pattern:** Confirm in `helm/argocd/values.yaml` that ArgoCD pods do not tolerate the control-plane `NoSchedule` taint.

### Namespace Model

All namespaces scaffolded by this substrate and available to downstream consumers:

| Namespace | Owner | Purpose |
|---|---|---|
| `argocd` | k3s substrate | ArgoCD GitOps control plane |
| `cert-manager` | k3s substrate | cert-manager controller |
| `external-secrets` | k3s substrate | External Secrets Operator |
| `metallb-system` | k3s substrate | MetalLB load balancer |
| `terminus-infra` | terminus/infra | Infra-owned workloads (secrets, postgres, etc.) |
| `terminus-platform` | terminus/platform | Platform-owned workloads (temporal, etc.) |

### Component Versions

Versions resolved to latest stable at story-write time. Pin in `versions.yaml` before each sprint:

| Component | Version strategy |
|---|---|
| ArgoCD | Latest stable at story-write time |
| cert-manager | Latest stable at story-write time |
| MetalLB | Latest stable at story-write time |
| External Secrets Operator | Latest stable at story-write time |
| Traefik | k3s bundled version (upgrade independently only if needed) |

### ESO → Vault Authentication

**Method: Kubernetes auth.** ESO uses a ServiceAccount JWT; Vault validates it against the cluster's token review API. Vault Kubernetes auth backend must be configured in `terminus-infra-secrets` as a pre-condition for ESO to function.

### Component Delivery Order (Implementation Sequence)

1. k3s bootstrap — first control-plane node (`--cluster-init`)
2. k3s HA join — second and third control-plane nodes (`--server`)
3. Worker nodes join (5 nodes)
4. Namespace + RBAC scaffolding
5. ArgoCD bootstrap (Ansible `helm install` — one-time)
6. MetalLB (required before any `LoadBalancer` service)
7. Traefik (needs MetalLB VIP for its `LoadBalancer` service)
8. cert-manager (needs Traefik for ingress; Vault must be reachable)
9. ESO (needs cert-manager TLS; needs Vault Kubernetes auth backend configured)
10. Consumer contract validation (all components healthy; downstream test deploy)

### Validation Result

**Architecture is implementation-ready.** All blocking gaps resolved. Structure, decisions, patterns, boundaries, and consumer contract are fully documented.
