# Technical Decisions Log

**Initiative:** terminus-infra-semaphoreui  
**Phase:** TechPlan  
**Date:** 2026-03-28  
**Status:** Approved for implementation planning

## Summary

This document captures the concrete technical decisions that govern implementation of the Semaphore UI deployment initiative. It is the execution-focused companion to `architecture.md` and serves as the authoritative decision record for manifest design, secret handling, database provisioning, GitOps delivery, and ingress configuration.

All decisions reflect answers provided in `techplan-questions.md` (all defaults accepted).

---

## Decision Register

### TD-001: Manifests Split Per-Resource

- **Status:** accepted
- **Decision:** Each Kubernetes resource kind gets its own YAML file (`deployment.yaml`, `service.yaml`, `pvc.yaml`, `externalsecret.yaml`, `certificate.yaml`, `ingress.yaml`).
- **Rationale:** Per-resource files are easier to read, diff, and review in pull requests. They also allow ArgoCD to report sync status at the individual resource level.
- **Consequences:**
  - The ArgoCD `Application` `path` points to the directory (`platforms/k3s/manifests/terminus-infra/semaphoreui/`), not a single file
  - All resources in the directory are synced together — no partial sync risk

---

### TD-002: Manifest Location — `platforms/k3s/manifests/terminus-infra/semaphoreui/`

- **Status:** accepted
- **Decision:** Semaphore UI k8s manifests live at `platforms/k3s/manifests/terminus-infra/semaphoreui/` in `terminus.infra`.
- **Rationale:** This path establishes the canonical pattern for namespace-scoped workloads: `manifests/{namespace}/{workload}/`. Future workloads follow the same convention.
- **Consequences:**
  - ArgoCD Application targets this path
  - Operators know where to look for any workload's raw manifests

---

### TD-003: Namespace — `terminus-infra` (existing)

- **Status:** accepted
- **Decision:** Deploy into the pre-existing `terminus-infra` namespace. Do not create a new namespace.
- **Rationale:** `terminus-infra` already exists and is the established home for infrastructure workloads. Creating a per-workload namespace adds operational overhead without benefit at this scale.
- **Consequences:**
  - All RBAC, NetworkPolicy, and ESO ClusterSecretStore bindings already scoped to this namespace apply

---

### TD-004: Image Tag — `latest` with `imagePullPolicy: Always`

- **Status:** accepted
- **Decision:** Use `semaphoreui/semaphore:latest` with `imagePullPolicy: Always`.
- **Rationale:** For a homelab deployment, `latest` minimizes operational overhead of tracking upstream releases. `imagePullPolicy: Always` is mandatory when using `latest` — without it, the locally cached image would be used and pod restarts would not pick up upstream updates.
- **Consequences:**
  - Pod restarts (including node eviction) will pull the latest upstream image
  - No pinned version means an upstream breaking change could affect the deployment on restart — acceptable risk for homelab
  - Future upgrade to pinned tags can be done by simply changing the tag in the Deployment manifest

---

### TD-005: Resource Requests and Limits

- **Status:** accepted
- **Decision:** Set resource requests of `100m` CPU / `256Mi` memory; set memory limit of `512Mi`; set no CPU limit.
- **Rationale:** Requests ensure the scheduler places the pod on a node with headroom. The memory limit prevents runaway memory consumption. No CPU limit avoids CPU throttling on an interactive web UI — CPU is not a constrained resource on this cluster for this workload.
- **Consequences:**
  - Pod OOMKill will occur if memory exceeds `512Mi` — increase limit if Semaphore grows (e.g., large task history)
  - No CPU throttling means Semaphore can burst CPU freely — monitor if cluster CPU pressure increases

---

### TD-006: Single Replica (replicas: 1)

- **Status:** accepted
- **Decision:** Deploy with `replicas: 1`.
- **Rationale:** The `local-path` StorageClass provisions `ReadWriteOnce` PVCs, which cannot be simultaneously mounted by multiple pods. Additionally, Semaphore UI does not support HA in a meaningful way with a single database connection pool and no stateless session architecture.
- **Consequences:**
  - Brief downtime during pod restarts (Deployment rolling update with `maxUnavailable: 1` is acceptable)
  - No redundancy — acceptable for homelab operations

---

### TD-007: PVC — `semaphoreui-data`, 1Gi, `local-path`, mounted at `/tmp/semaphore`

- **Status:** accepted
- **Decision:** PVC named `semaphoreui-data`, `1Gi`, `StorageClass: local-path`, mounted at `/tmp/semaphore` (Semaphore default data directory).
- **Rationale:** `/tmp/semaphore` is Semaphore's default data directory — no `SEMAPHORE_TMP_PATH` override required, minimizing configuration surface. `1Gi` exceeds the expected usage for task logs and local state at homelab scale.
- **Consequences:**
  - If the PVC mount path is changed in a future Semaphore version, the manifest will need updating
  - `local-path` binds to the scheduling node; pod + node affinity docs should note this constraint

---

### TD-008: Vault KV Path — `secret/terminus/infra/semaphoreui`

- **Status:** accepted
- **Decision:** All Semaphore credentials stored at `secret/terminus/infra/semaphoreui` (KV v2).
- **Rationale:** Using `semaphoreui` (not `semaphore`) is more consistent with the initiative name and avoids potential path collision with a future standalone Semaphore non-UI component. Single path keeps the secret policy simple.
- **Note:** The PRD referenced `secret/terminus/infra/semaphore/` — this architecture supersedes that reference. Implementation stories must use `semaphoreui`.
- **Consequences:**
  - Vault policy must grant ESO service account read access to `secret/data/terminus/infra/semaphoreui`
  - Operator must seed this path before deployment (story prerequisite)

---

### TD-009: ExternalSecret — Single Secret, Five Fields

- **Status:** accepted
- **Decision:** A single `ExternalSecret` syncs five Vault fields to one k8s Secret named `semaphoreui-secrets`.
- **Fields:**

  | Vault Field | k8s Secret Key |
  |---|---|
  | `admin_user` | `SEMAPHORE_ADMIN` |
  | `admin_password` | `SEMAPHORE_ADMIN_PASSWORD` |
  | `admin_email` | `SEMAPHORE_ADMIN_EMAIL` |
  | `db_password` | `SEMAPHORE_DB_PASS` |
  | `access_key_encryption` | `SEMAPHORE_ACCESS_KEY_ENCRYPTION` |

- **Rationale:** A single ExternalSecret and single k8s Secret minimize the number of resources to manage and simplify the Deployment's `envFrom` reference. Keeping db_password co-located avoids a second ExternalSecret and a separate Vault policy grant.
- **Consequences:**
  - If Semaphore adds additional required credentials in a future version, the ExternalSecret must be updated

---

### TD-010: Postgres Connection — Hardcoded Non-Secret Env Vars, Secret for Password Only

- **Status:** accepted
- **Decision:** Inject `SEMAPHORE_DB_HOST`, `SEMAPHORE_DB_PORT`, `SEMAPHORE_DB_NAME`, `SEMAPHORE_DB_USER` as plain env vars in the Deployment manifest. Inject `SEMAPHORE_DB_PASS` from `semaphoreui-secrets`.
- **Values:**
  - `SEMAPHORE_DB_DIALECT`: `postgres`
  - `SEMAPHORE_DB_HOST`: `10.0.0.56`
  - `SEMAPHORE_DB_PORT`: `5432`
  - `SEMAPHORE_DB_NAME`: `semaphore`
  - `SEMAPHORE_DB_USER`: `semaphore`
- **Rationale:** The host, port, db name, and user are not sensitive and do not need Vault-managed rotation. Hardcoding them in the manifest makes the connection intent explicit and auditable in git. Only the password is sensitive and must come from the secret.
- **Consequences:**
  - If the Patroni primary IP changes (failover to different node with different VIP), the manifest must be updated, OR a stable DNS name/VIP must be used — this is a known constraint of direct-IP postgres binding

---

### TD-011: Database — Name `semaphore`, Role `semaphore`

- **Status:** accepted
- **Decision:** Postgres database name: `semaphore`. Postgres role name: `semaphore`.
- **Rationale:** Name parity between database and role is a common, clean convention. Consistent with other modules in the repo.
- **Consequences:**
  - Role `semaphore` has full access to database `semaphore` only
  - Role password matches `semaphoreui.db_password` in Vault

---

### TD-012: Tofu — Standalone Environment at `tofu/environments/postgres/semaphore/`

- **Status:** accepted
- **Decision:** Create a standalone OpenTofu environment at `tofu/environments/postgres/semaphore/` using the `postgres-databases` module. Do not add semaphore to an existing environment.
- **Rationale:** Keeping concerns separate (one Tofu env per database consumer) follows the established `postgres/dev/` pattern. Mixing app database provisioning into the cluster bootstrap environment would create dependency coupling.
- **Tofu state key:** `tofu/postgres/semaphore` (Consul backend, per platform standard)
- **Consequences:**
  - Operators run `tofu apply` in `tofu/environments/postgres/semaphore/` independently
  - DB password passed as a variable at apply time — never hardcoded or committed

---

### TD-013: Ingress Hostname — `semaphore.trantor.internal`

- **Status:** accepted
- **Decision:** Semaphore UI is accessible at `https://semaphore.trantor.internal`.
- **Consequences:**
  - DNS A-record required: `semaphore.trantor.internal → 10.0.0.126`
  - Certificate SAN must include `semaphore.trantor.internal`

---

### TD-014: Explicit Certificate Object (not annotation-driven)

- **Status:** accepted
- **Decision:** Use an explicit `Certificate` resource (`certificate.yaml`) rather than cert-manager ingress annotations.
- **Rationale:** Explicit Certificate objects are independently inspectable (`kubectl get certificate`), have their own status conditions (e.g., `Ready`, `Issuing`), and decouple cert lifecycle from the Ingress resource. Debugging cert issues is easier with an explicit object.
- **Consequences:**
  - Two files for TLS: `certificate.yaml` + `ingress.yaml`
  - `certificate.yaml` must be committed and synced before the Ingress references the TLS secret

---

### TD-015: TLS Secret Name — `semaphoreui-tls`

- **Status:** accepted
- **Decision:** cert-manager Certificate writes to k8s Secret `semaphoreui-tls`.
- **Rationale:** Namespaced, workload-scoped name. Avoids collision with ArgoCD TLS secret.
- **Consequences:**
  - Ingress `spec.tls[].secretName` must reference `semaphoreui-tls`

---

### TD-016: ArgoCD Sync Policy — Automated, Self-Heal, Prune

- **Status:** accepted
- **Decision:** ArgoCD Application syncs automatically with `selfHeal: true` and `prune: true`.
- **Rationale:** This is the GitOps reference pattern. Git is the source of truth. Manual kubectl changes are corrected automatically. Resources removed from git are removed from the cluster.
- **Consequences:**
  - Manual `kubectl` changes to managed resources will be reverted by ArgoCD within the sync interval
  - Operators must commit changes to git, not apply them directly

---

### TD-017: ArgoCD Tracks `main` Branch (HEAD)

- **Status:** accepted
- **Decision:** ArgoCD `Application` `targetRevision: main` (HEAD tracking).
- **Rationale:** HEAD tracking is appropriate for homelab environments. Reduces deployment ceremony — no tag-bumping workflow required for routine updates.
- **Consequences:**
  - Any commit to `main` in `terminus.infra` within the `semaphoreui` manifest path will be auto-applied
  - For break-glass, disable automated sync in ArgoCD UI before pushing experimental changes

---

### TD-018: ArgoCD Insecure Mode via `argocd-cmd-params-cm`

- **Status:** accepted
- **Decision:** Configure ArgoCD to run in insecure mode (plain HTTP internally) by setting `server.insecure: "true"` in the `argocd-cmd-params-cm` ConfigMap in the `argocd` namespace. Do not modify the ArgoCD Deployment spec directly.
- **Rationale:** ConfigMap-based configuration survives ArgoCD upgrades without Deployment spec drift. `--insecure` flag is the canonical way to configure this per ArgoCD documentation.
- **Consequences:**
  - `argocd-server` Deployment must be rolled after the ConfigMap is applied: `kubectl rollout restart deployment argocd-server -n argocd`
  - ArgoCD UI redirects to HTTPS at the Traefik layer; cluster-internal traffic between Traefik and argocd-server is plain HTTP

---

### TD-019: ArgoCD Ingress Hostname — `argocd.trantor.internal`

- **Status:** accepted
- **Decision:** ArgoCD is accessible at `https://argocd.trantor.internal`.
- **Consequences:**
  - DNS A-record required: `argocd.trantor.internal → 10.0.0.126`
  - Separate Certificate and Ingress manifests required in `platforms/k3s/manifests/argocd/`

---

### TD-020: ArgoCD Ingress Manifests at `platforms/k3s/manifests/argocd/`

- **Status:** accepted
- **Decision:** ArgoCD ingress resources (`ingress.yaml`, `certificate.yaml`) live at `platforms/k3s/manifests/argocd/` — separate from the App-of-Apps directory.
- **Rationale:** The ArgoCD ingress cannot be delivered by an ArgoCD Application (bootstrapping problem — ArgoCD must first be accessible). Placing them separately makes the manual-apply nature explicit and prevents accidental App-of-Apps inclusion.
- **Consequences:**
  - These manifests are applied via `kubectl apply` (not via ArgoCD sync)
  - Document in runbook to distinguish from GitOps-managed resources
