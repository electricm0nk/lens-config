---
phase: devproposal
initiative: terminus-infra-semaphoreui
date: 2026-03-28
status: approved
epics: [1, 2, 3]
stories: [1-1, 1-2, 2-1, 2-2, 2-3, 3-1]
---

# terminus-infra-semaphoreui — Epic & Story Breakdown

## Requirements Inventory

### Functional Requirements

FR-01.1: A Kubernetes `Deployment` runs `semaphoreui/semaphore:latest` in the `terminus-infra` namespace
FR-01.2: The deployment mounts a `1Gi` PVC (`local-path` StorageClass) for `/tmp/semaphore`
FR-01.3: Admin credentials and database connection details read from k8s Secret `semaphoreui-secrets`
FR-01.4: A `ClusterIP` Service exposes the Semaphore pod on port `3000`
FR-01.5: Readiness probe ensures traffic is only routed to a healthy pod
FR-02.1: Traefik `Ingress` routes `https://semaphore.trantor.internal` to the Semaphore Service
FR-02.2: cert-manager `Certificate` issued by ClusterIssuer `vault-pki` bound to Semaphore Ingress
FR-02.3: HTTP→HTTPS redirect enforced (platform-level via Traefik IngressClass)
FR-03.1: Vault KV path `secret/terminus/infra/semaphoreui` holds all credentials
FR-03.2: ESO `ExternalSecret` in `terminus-infra` namespace syncs Vault path to `semaphoreui-secrets`
FR-03.3: No plaintext credentials in any committed manifest
FR-04.1: Postgres database `semaphore` and role `semaphore` provisioned via `postgres-databases` Tofu module
FR-04.2: DB password stored at Vault path, referenced by ExternalSecret
FR-04.3: Tofu state at `tofu/environments/postgres/semaphore/` in `terminus.infra`
FR-05.1: ArgoCD `Application` in `platforms/k3s/argocd/apps/semaphoreui.yaml` targets the Semaphore manifest path
FR-05.2: Application syncs from `main` branch, deploys to `terminus-infra`
FR-05.3: Sync policy: automated, prune=true, selfHeal=true
FR-06.1: Traefik `Ingress` routes `https://argocd.trantor.internal` to `argocd-server` Service
FR-06.2: cert-manager `Certificate` issued for `argocd.trantor.internal` by `vault-pki`
FR-06.3: ArgoCD server configured to run insecure (plain HTTP) via `argocd-cmd-params-cm`

### Non-Functional Requirements

NFR-01.1: No credentials or tokens in any committed file
NFR-01.2: All HTTPS endpoints use `vault-pki` issued certificates
NFR-01.3: Semaphore Service is ClusterIP only — no direct external exposure
NFR-02.1: Pod restarts automatically on failure (default Deployment restart policy)
NFR-02.2: PVC must be bound prior to pod scheduling
NFR-03.1: ArgoCD Application health visible in ArgoCD UI
NFR-04.1: Manifests at `platforms/k3s/manifests/terminus-infra/semaphoreui/` (per-resource split)
NFR-04.2: Tofu postgres DB provisioning follows `postgres-databases` module pattern

### FR Coverage Map

| Epic | FRs Covered | NFRs Covered |
|---|---|---|
| Epic 1: Data Foundation | FR-03, FR-04 | NFR-01.1, NFR-01.3 |
| Epic 2: Semaphore Application Delivery | FR-01, FR-02, FR-05 | NFR-01.2, NFR-02, NFR-03.1, NFR-04 |
| Epic 3: ArgoCD Platform Access | FR-06 | NFR-01.2 |

---

## Epic List

1. Epic 1 — Data Foundation (Postgres + Vault Secrets)
2. Epic 2 — Semaphore Application Delivery
3. Epic 3 — ArgoCD Platform Access

---

## Epic 1: Data Foundation (Postgres + Vault Secrets)

Provision the database and secret infrastructure required for Semaphore UI to start. All secrets are sourced from Vault and injected via ESO — no plaintext credentials exist in any manifest. This epic is a prerequisite for Epic 2.

**Status:** ready

### Story 1.1: Provision Semaphore Postgres Database and Role via OpenTofu

**Status:** ready
**Epic:** 1

As a platform operator,
I want a dedicated `semaphore` Postgres database and role provisioned in the Terminus Patroni cluster,
So that Semaphore UI has an isolated database with appropriate access credentials.

**Acceptance Criteria:**

**Given** the Patroni Postgres HA cluster is operational at `10.0.0.56:5432`
**When** `tofu apply` runs in `tofu/environments/postgres/semaphore/` with the DB password variable sourced from Vault
**Then** database `semaphore` exists in the Patroni cluster
**And** role `semaphore` exists with `GRANT ALL` on the `semaphore` database
**And** Tofu state is stored in Consul at key `tofu/postgres/semaphore`
**And** no credentials appear in any committed Tofu file (password passed via variable at apply time)

**Tasks:**
- [ ] Task 1: Create `tofu/environments/postgres/semaphore/` in `terminus.infra`
  - [ ] Create `backend.tf` using Consul state backend key `tofu/postgres/semaphore`
  - [ ] Create `variables.tf` with `db_password` variable (type `string`, sensitive=true)
  - [ ] Create `main.tf` calling the `postgres-databases` module with db=`semaphore`, role=`semaphore`
- [ ] Task 2: Run Tofu provisioning
  - [ ] `tofu init` — verify remote state backend connects
  - [ ] `tofu plan -var="db_password=..."` — review plan output
  - [ ] `tofu apply` — confirm DB and role created
- [ ] Task 3: Verify database accessible
  - [ ] Connect via `psql -h 10.0.0.56 -U semaphore semaphore` and confirm connection succeeds

**Implementation Notes:**
- The DB password used at apply time MUST match the value stored in Vault at `secret/terminus/infra/semaphoreui` field `db_password`
- The existing `postgres/dev/` environment manages the cluster itself — do NOT modify it
- Consul backend config: reference existing pattern in the repo

---

### Story 1.2: Seed Vault KV and Create ESO ExternalSecret

**Status:** ready
**Epic:** 1

As a platform operator,
I want Semaphore UI's credentials stored in Vault and synced to a Kubernetes Secret via ESO,
So that the Semaphore Deployment can read all required credentials without any plaintext secrets in manifests.

**Acceptance Criteria:**

**Given** Vault is operational at `vault.trantor.internal:8200` with KV v2 at `secret/`
**When** the operator seeds `secret/terminus/infra/semaphoreui` with the 5 required fields AND the ESO `ExternalSecret` manifest is committed and applied (via ArgoCD or kubectl)
**Then** k8s Secret `semaphoreui-secrets` exists in the `terminus-infra` namespace
**And** the Secret contains keys: `SEMAPHORE_ADMIN`, `SEMAPHORE_ADMIN_PASSWORD`, `SEMAPHORE_ADMIN_EMAIL`, `SEMAPHORE_DB_PASS`, `SEMAPHORE_ACCESS_KEY_ENCRYPTION`
**And** `kubectl get externalsecret semaphoreui-externalsecret -n terminus-infra` shows `STATUS: SecretSynced`

**Tasks:**
- [ ] Task 1: Seed Vault KV (operator manual prerequisite)
  - [ ] Confirm `SEMAPHORE_ACCESS_KEY_ENCRYPTION` is a 32+ character random string
  - [ ] Run: `vault kv put secret/terminus/infra/semaphoreui admin_user=<...> admin_password=<...> admin_email=<...> db_password=<...> access_key_encryption=<...>`
  - [ ] Verify with `vault kv get secret/terminus/infra/semaphoreui`
- [ ] Task 2: Create `externalsecret.yaml` manifest
  - [ ] File path: `platforms/k3s/manifests/terminus-infra/semaphoreui/externalsecret.yaml`
  - [ ] `spec.secretStoreRef.name: vault-backend`, `kind: ClusterSecretStore`
  - [ ] `spec.target.name: semaphoreui-secrets`
  - [ ] Map all 5 Vault fields to the corresponding `SEMAPHORE_*` environment variable keys
- [ ] Task 3: Verify ExternalSecret sync
  - [ ] Apply manifest (kubectl or ArgoCD) and confirm `SecretSynced` status
  - [ ] `kubectl get secret semaphoreui-secrets -n terminus-infra` — confirm 5 keys present

**Implementation Notes:**
- Vault field `db_password` MUST have the same value used in Story 1.1 Tofu apply
- `access_key_encryption` is used by Semaphore to encrypt stored Ansible access keys at rest — generate with `openssl rand -base64 32`
- ESO ClusterSecretStore `vault-backend` already exists from k3s platform setup
- refreshInterval on ExternalSecret: `15m` is reasonable for homelab

---

## Epic 2: Semaphore Application Delivery

Deploy Semaphore UI as a fully GitOps-managed workload with TLS ingress, registered as an ArgoCD Application. All resource manifests are committed to `terminus.infra` under `platforms/k3s/manifests/terminus-infra/semaphoreui/`. Depends on Epic 1 completion (secret and database must exist before pod starts).

**Status:** ready

### Story 2.1: Create Semaphore Deployment, Service, and PVC Manifests

**Status:** ready
**Epic:** 2

As a platform operator,
I want the Semaphore Deployment, Service, and PersistentVolumeClaim manifests committed to `terminus.infra`,
So that ArgoCD can deliver the Semaphore pod to the cluster with persistent storage and correct credentials.

**Acceptance Criteria:**

**Given** the `semaphoreui-secrets` k8s Secret is synced (Story 1.2 complete) AND the `semaphore` Postgres DB exists (Story 1.1 complete)
**When** the manifests are committed and ArgoCD syncs them to the cluster
**Then** a `semaphoreui` pod is Running in the `terminus-infra` namespace
**And** PVC `semaphoreui-data` is Bound and mounted at `/tmp/semaphore`
**And** `kubectl logs -n terminus-infra deployment/semaphoreui` shows no fatal errors
**And** `kubectl exec -n terminus-infra deployment/semaphoreui -- wget -q -O- http://localhost:3000` returns HTTP 200 or 302

**Tasks:**
- [ ] Task 1: Create `pvc.yaml`
  - [ ] name: `semaphoreui-data`, namespace: `terminus-infra`
  - [ ] storageClassName: `local-path`, accessModes: `[ReadWriteOnce]`, capacity: `1Gi`
- [ ] Task 2: Create `deployment.yaml`
  - [ ] image: `semaphoreui/semaphore:latest`, imagePullPolicy: `Always`, replicas: `1`
  - [ ] volumeMount: `/tmp/semaphore` from `semaphoreui-data` PVC
  - [ ] resources: requests `100m/256Mi`, limits memory `512Mi` (no CPU limit)
  - [ ] envFrom: `semaphoreui-secrets` secretRef (all `SEMAPHORE_*` vars)
  - [ ] Plain env vars: `SEMAPHORE_DB_DIALECT=postgres`, `SEMAPHORE_DB_HOST=10.0.0.56`, `SEMAPHORE_DB_PORT=5432`, `SEMAPHORE_DB_NAME=semaphore`, `SEMAPHORE_DB_USER=semaphore`
  - [ ] Readiness probe: HTTP GET `/api/ping` port `3000`, initialDelaySeconds `15`, periodSeconds `10`
- [ ] Task 3: Create `service.yaml`
  - [ ] name: `semaphoreui`, namespace: `terminus-infra`, type: `ClusterIP`, port `3000` → targetPort `3000`
- [ ] Task 4: Verify deployment health
  - [ ] Confirm pod Running, PVC Bound, no crash loops

**Implementation Notes:**
- Actual Semaphore health endpoint: verify `/api/ping` returns `{"message":"PONG"}` — check upstream docs if different
- DB connection (SEMAPHORE_DB_PASS) is injected via secretKeyRef from `semaphoreui-secrets` — use explicit env var reference (not envFrom) for this one field if envFrom merges cause confusion
- `local-path` PVC binds to scheduling node — note in story acceptance that pod/node affinity is acceptable for homelab
- Add labels: `app.kubernetes.io/name: semaphoreui`, `app.kubernetes.io/part-of: terminus-infra`

---

### Story 2.2: Create Semaphore Ingress and cert-manager Certificate

**Status:** ready
**Epic:** 2

As a platform operator,
I want HTTPS access to Semaphore UI at `https://semaphore.trantor.internal` with a valid cert-manager-issued TLS certificate,
So that the Semaphore web interface is accessible from the homelab network with proper TLS.

**Acceptance Criteria:**

**Given** the Semaphore Deployment is Running (Story 2.1 complete) AND DNS A-record `semaphore.trantor.internal → 10.0.0.126` exists
**When** the Certificate and Ingress manifests are applied (via ArgoCD)
**Then** `kubectl get certificate semaphoreui-tls-cert -n terminus-infra` shows `READY: True`
**And** `curl -sk https://semaphore.trantor.internal | grep -i semaphore` returns content
**And** `curl -v https://semaphore.trantor.internal 2>&1 | grep "issuer"` shows the Vault PKI issuer CN

**Tasks:**
- [ ] Task 1: Add DNS record (operator prerequisite)
  - [ ] Synology DNS: add A-record `semaphore.trantor.internal → 10.0.0.126`
  - [ ] Verify with `dig semaphore.trantor.internal`
- [ ] Task 2: Create `certificate.yaml`
  - [ ] name: `semaphoreui-tls-cert`, namespace: `terminus-infra`
  - [ ] issuerRef: kind `ClusterIssuer`, name `vault-pki`
  - [ ] dnsNames: `[semaphore.trantor.internal]`
  - [ ] secretName: `semaphoreui-tls`
- [ ] Task 3: Create `ingress.yaml`
  - [ ] name: `semaphoreui`, namespace: `terminus-infra`, ingressClassName: `traefik`
  - [ ] spec.tls[]: host `semaphore.trantor.internal`, secretName `semaphoreui-tls`
  - [ ] rules[]: host `semaphore.trantor.internal` → service `semaphoreui` port `3000`
- [ ] Task 4: Verify end-to-end TLS
  - [ ] Wait for cert issuance (check Certificate resource events if slow)
  - [ ] Test: `curl -sk https://semaphore.trantor.internal` returns content

**Implementation Notes:**
- The ClusterIssuer `vault-pki` must be healthy before applying — check with `kubectl get clusterissuer vault-pki`
- HTTP→HTTPS redirect handled by Traefik platform defaults — no per-ingress annotation needed
- If cert issuance stalls: `kubectl describe certificate semaphoreui-tls-cert -n terminus-infra` and check CertificateRequest events

---

### Story 2.3: Register Semaphore ArgoCD Application

**Status:** ready
**Epic:** 2

As a platform operator,
I want an ArgoCD Application resource that tracks the Semaphore manifests from `terminus.infra` `main` branch,
So that all future changes to Semaphore manifests in git are automatically applied to the cluster via GitOps.

**Acceptance Criteria:**

**Given** the Semaphore manifests are committed at `platforms/k3s/manifests/terminus-infra/semaphoreui/` in `terminus.infra` `main`
**When** the `semaphoreui.yaml` ArgoCD Application is applied (via App-of-Apps or kubectl)
**Then** `kubectl get application semaphoreui -n argocd` shows `SYNC STATUS: Synced, HEALTH STATUS: Healthy`
**And** all Semaphore resources (Deployment, Service, PVC, ExternalSecret, Certificate, Ingress) appear in ArgoCD UI as managed
**And** a test commit to the manifest path in `terminus.infra` is auto-applied within the ArgoCD sync interval

**Tasks:**
- [ ] Task 1: Create `platforms/k3s/argocd/apps/semaphoreui.yaml` in `terminus.infra`
  - [ ] `spec.source.repoURL`: `terminus.infra` repo URL
  - [ ] `spec.source.targetRevision: main`
  - [ ] `spec.source.path: platforms/k3s/manifests/terminus-infra/semaphoreui`
  - [ ] `spec.destination.namespace: terminus-infra`
  - [ ] `spec.syncPolicy.automated: {selfHeal: true, prune: true}`
- [ ] Task 2: Verify ArgoCD picks up and syncs the Application
  - [ ] ArgoCD App-of-Apps adds `semaphoreui` to the managed app list (may need App-of-Apps sync)
  - [ ] All Semaphore resources show green in ArgoCD UI
- [ ] Task 3: Smoke test — validate full deployment
  - [ ] `curl -sk https://semaphore.trantor.internal` — confirm 200/redirect to login
  - [ ] Log in to Semaphore at `https://semaphore.trantor.internal` with Vault-sourced admin credentials
  - [ ] Confirm admin user can access the dashboard

**Implementation Notes:**
- If the App-of-Apps does not auto-sync new Application files in `apps/`, manually trigger sync on the root app: `kubectl patch application root-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'`
- Ensure the `semaphoreui` Application `project: default` — unless a dedicated ArgoCD project exists for workloads

---

## Epic 3: ArgoCD Platform Access

Expose the ArgoCD UI at `https://argocd.trantor.internal` via Traefik ingress with TLS. ArgoCD must first be configured to run in insecure mode (plain HTTP internally) so that Traefik can terminate TLS at the edge without a redirect loop. ArgoCD ingress manifests are applied via `kubectl` (not via ArgoCD itself, due to the bootstrapping problem).

**Status:** ready

### Story 3.1: Configure ArgoCD Insecure Mode and Deploy ArgoCD Ingress

**Status:** ready
**Epic:** 3

As a platform operator,
I want the ArgoCD UI accessible at `https://argocd.trantor.internal` with TLS,
So that ArgoCD can be used from the homelab network browser without port-forwarding.

**Acceptance Criteria:**

**Given** ArgoCD is operational (ClusterIP only) AND DNS A-record `argocd.trantor.internal → 10.0.0.126` exists AND cert-manager ClusterIssuer `vault-pki` is healthy
**When** the `argocd-cmd-params-cm` ConfigMap patch is applied, argocd-server is restarted, and the Certificate + Ingress manifests are applied
**Then** `kubectl get certificate argocd-tls-cert -n argocd` shows `READY: True`
**And** `curl -sk https://argocd.trantor.internal` returns ArgoCD UI content (HTML with "Argo CD")
**And** the ArgoCD web interface at `https://argocd.trantor.internal` loads the login page without certificate warnings in a browser that trusts the Vault PKI CA
**And** ArgoCD continues to sync other applications normally (no regression from insecure mode)

**Tasks:**
- [ ] Task 1: Add DNS record (operator prerequisite)
  - [ ] Synology DNS: add A-record `argocd.trantor.internal → 10.0.0.126`
  - [ ] Verify with `dig argocd.trantor.internal`
- [ ] Task 2: Patch `argocd-cmd-params-cm` ConfigMap
  - [ ] Create `platforms/k3s/manifests/argocd/configmap-argocd-cmd-params.yaml` in `terminus.infra` (or apply directly)
  - [ ] Set `server.insecure: "true"` in the ConfigMap
  - [ ] Apply: `kubectl apply -f platforms/k3s/manifests/argocd/configmap-argocd-cmd-params.yaml`
- [ ] Task 3: Restart ArgoCD server to activate insecure mode
  - [ ] `kubectl rollout restart deployment argocd-server -n argocd`
  - [ ] Confirm rollout completes: `kubectl rollout status deployment argocd-server -n argocd`
- [ ] Task 4: Create `platforms/k3s/manifests/argocd/certificate.yaml`
  - [ ] name: `argocd-tls-cert`, namespace: `argocd`
  - [ ] issuerRef: kind `ClusterIssuer`, name `vault-pki`
  - [ ] dnsNames: `[argocd.trantor.internal]`
  - [ ] secretName: `argocd-tls`
- [ ] Task 5: Create `platforms/k3s/manifests/argocd/ingress.yaml`
  - [ ] name: `argocd`, namespace: `argocd`, ingressClassName: `traefik`
  - [ ] spec.tls[]: host `argocd.trantor.internal`, secretName `argocd-tls`
  - [ ] rules[]: host `argocd.trantor.internal` → service `argocd-server` port `80` (plain HTTP, insecure mode)
- [ ] Task 6: Apply ArgoCD manifests via kubectl (NOT via ArgoCD)
  - [ ] `kubectl apply -f platforms/k3s/manifests/argocd/certificate.yaml`
  - [ ] `kubectl apply -f platforms/k3s/manifests/argocd/ingress.yaml`
  - [ ] Monitor cert issuance: `kubectl get certificate argocd-tls-cert -n argocd -w`
- [ ] Task 7: Validate ArgoCD access
  - [ ] `curl -sk https://argocd.trantor.internal | grep -i "argo"` returns content
  - [ ] Log in to ArgoCD via browser at `https://argocd.trantor.internal`
  - [ ] Confirm all existing ArgoCD Applications remain healthy

**Implementation Notes:**
- The ArgoCD manifests at `platforms/k3s/manifests/argocd/` are intentionally NOT managed by an ArgoCD Application — manually applied to avoid the bootstrapping problem (ArgoCD can't apply its own ingress if it's not yet accessible)
- After insecure mode is active, ArgoCD redirects HTTP→HTTPS at the Traefik layer — cluster-internal traffic is plain HTTP (acceptable, cluster network is trusted)
- If you see a redirect loop after applying: check that `spec.backend.service.port` in the Ingress is `80` (not `443`) and that `argocd-cmd-params-cm` correctly has `server.insecure: "true"`
- Consider adding a comment in the manifest noting the manual-apply requirement to prevent future operators from incorrectly adding it to the App-of-Apps
