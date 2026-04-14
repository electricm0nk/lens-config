---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - docs/terminus/infra/k3s/product-brief.md
  - docs/terminus/infra/k3s/prd.md
  - docs/terminus/infra/k3s/architecture.md
project_name: terminus-infra-k3s
---

# terminus-infra-k3s — Epic Breakdown

## Requirements Inventory

### Functional Requirements

FR1: k3s cluster bootstrap — first control-plane node initialized with embedded etcd (`--cluster-init`)
FR2: k3s HA join — second and third control-plane nodes joined to form 3-node etcd quorum
FR3: Worker node join — 5 worker nodes joined and accepting workload scheduling
FR4: Node scheduling enforced — control-plane nodes tainted `NoSchedule`; workloads schedule on workers only
FR5: Namespace scaffolding — all 6 namespaces created (`argocd`, `cert-manager`, `external-secrets`, `metallb-system`, `terminus-infra`, `terminus-platform`)
FR6: RBAC baseline — ClusterRoles and ServiceAccounts created per naming conventions
FR7: ArgoCD bootstrap — ArgoCD installed via Ansible `helm install`; App-of-Apps root app deployed
FR8: ArgoCD takes over — all remaining cluster components managed via ArgoCD `Application` resources
FR9: MetalLB deployed — L2 mode, IP pool `10.0.0.126–10.0.0.254` configured
FR10: Traefik configured — `LoadBalancer` service assigned MetalLB VIP; `IngressClass: traefik` active
FR11: cert-manager deployed — `ClusterIssuer` configured pointing at `vault.trantor.internal` (Vault PKI)
FR12: ESO deployed — External Secrets Operator installed; `SecretStore` configured with Kubernetes auth against Vault
FR13: ESO bootstrap secret — Vault Kubernetes auth backend configured (pre-condition from `terminus-infra-secrets`); ESO ServiceAccount JWT registered
FR14: Consumer contract validated — all 8 contract items verifiable (nodes Ready, namespaces present, ESO syncing, cert-manager issuing, MetalLB pool active, Traefik healthy, ArgoCD synced, storage class set)
FR15: Cluster upgrade workflow — Ansible `upgrade.yml` playbook documented and validated against a test upgrade run
FR16: Node replacement workflow — Ansible `node-replace.yml` playbook documented and validated (drain, remove, rejoin)
FR17: Bootstrap documentation — `runbook-bootstrap.md` authored and operator-validated
FR18: Upgrade documentation — `runbook-upgrade.md` authored
FR19: Node replacement documentation — `runbook-node-replace.md` authored

### Non-Functional Requirements

NFR1: Repeatability — cluster bootstrap 100% reproducible from committed artifacts with no untracked manual steps
NFR2: Security — zero secrets or credentials committed to source control; all runtime secrets via Vault + ESO
NFR3: Recoverability — node replacement and cluster rebuild workflows documented and operator-validated before feature is considered complete
NFR4: Consumer contract stability — all 8 downstream contract items explicit, testable, and documented
NFR5: Bootstrap ordering guarantee — "cluster ready" is defined as a testable state, not prose; consumer contract items are checkable conditions
NFR6: No hardcoded IPs in Ansible — all host references via inventory group names
NFR7: No anonymous Ansible tasks — all tasks have a `name:` attribute
NFR8: Standard labels on all managed resources — `app.kubernetes.io/managed-by`, `app.kubernetes.io/part-of`, `terminus.io/domain`, `terminus.io/service`
NFR9: One `values.yaml` per Helm component — no inline `--set` in Ansible or ArgoCD specs
NFR10: ArgoCD `syncPolicy.automated.prune: true, selfHeal: true` on all Application resources

### Additional Technical Requirements (from Architecture)

- k3s install via Ansible `k3s-server` and `k3s-agent` roles (no k3sup or separate binary)
- ArgoCD App-of-Apps structure: `root-app.yaml` → `argocd/apps/*.yaml`
- Helm values files committed at `infra/k3s/helm/{component}/values.yaml`
- ArgoCD manifests at `infra/k3s/argocd/`
- Kubernetes manifests at `infra/k3s/manifests/`
- Ansible playbooks at `infra/k3s/ansible/`
- Component versions: latest stable at story-write time; pin in implementation
- ESO Vault auth method: Kubernetes auth (ServiceAccount JWT)
- MetalLB: `IPAddressPool` + `L2Advertisement` for `10.0.0.126–10.0.0.254`
- cert-manager ClusterIssuer: Vault PKI issuer pointing at `vault.trantor.internal`
- ArgoCD and ESO pods must NOT tolerate `NoSchedule` control-plane taint

### FR Coverage Map

| Epic | FRs Covered | NFRs Covered |
|---|---|---|
| Epic 1: Cluster Bootstrap & Topology | FR1–FR4 | NFR1, NFR2, NFR6, NFR7 |
| Epic 2: Cluster Foundation (ArgoCD + Namespaces + RBAC) | FR5–FR8 | NFR8, NFR9, NFR10 |
| Epic 3: Network & Ingress (MetalLB + Traefik) | FR9–FR10 | NFR8, NFR9 |
| Epic 4: TLS & Secrets Infrastructure (cert-manager + ESO) | FR11–FR13 | NFR2, NFR8, NFR9 |
| Epic 5: Consumer Contract Validation | FR14 | NFR4, NFR5 |
| Epic 6: Operational Lifecycle (Upgrade + Node Replace + Docs) | FR15–FR19 | NFR3 |

## Epic List

1. Epic 1 — Cluster Bootstrap & Topology
2. Epic 2 — Cluster Foundation (ArgoCD + Namespaces + RBAC)
3. Epic 3 — Network & Ingress (MetalLB + Traefik)
4. Epic 4 — TLS & Secrets Infrastructure (cert-manager + ESO)
5. Epic 5 — Consumer Contract Validation
6. Epic 6 — Operational Lifecycle (Upgrade + Node Replace + Docs)

---

## Epic 1: Cluster Bootstrap & Topology

Initialize a 3-node HA k3s control plane and join 5 worker nodes via Ansible, with control-plane nodes tainted NoSchedule so all workloads schedule exclusively on dedicated workers. Kubeconfig accessible from admin workstation at completion.

### Story 1.1: Bootstrap First Control-Plane Node

As an operator,
I want the first control-plane node initialized via Ansible with embedded etcd,
So that a k3s cluster foundation exists and kubeconfig is accessible from my workstation.

**Acceptance Criteria:**

**Given** `infra/k3s/ansible/inventory/hosts.yml` defines a `k3s_control_plane` group using hostnames (no hardcoded IPs)
**When** `ansible-playbook bootstrap.yml` runs against the first control-plane host
**Then** k3s is installed with `--cluster-init`, embedded etcd is running, and kubeconfig is downloaded to `~/.kube/config` on the admin workstation
**And** `kubectl get nodes` returns 1 node in `Ready` state with `control-plane` role
**And** no k3s token or credential is present in any committed file (generated at runtime only)
**And** all Ansible tasks in the playbook have a `name:` attribute

### Story 1.2: Join Second and Third Control-Plane Nodes

As an operator,
I want the second and third control-plane nodes joined to form a 3-member etcd quorum,
So that the control plane is highly available and tolerates a single node failure.

**Acceptance Criteria:**

**Given** Story 1.1 is complete (first control-plane running, kubeconfig accessible)
**When** `ansible-playbook join-control-plane.yml` runs against control-plane nodes 2 and 3
**Then** `kubectl get nodes` shows 3 nodes with `control-plane` role in `Ready` state
**And** `etcdctl member list` reports 3 healthy members
**And** all host references in the playbook use inventory group names, not IP literals

### Story 1.3: Join Worker Nodes

As an operator,
I want 5 worker nodes joined to the cluster via Ansible,
So that dedicated worker capacity is available for workload scheduling.

**Acceptance Criteria:**

**Given** a 3-node HA control plane from Story 1.2
**When** `ansible-playbook join-workers.yml` runs against the `k3s_workers` inventory group
**Then** `kubectl get nodes` shows 5 additional nodes in `Ready` state with worker labels
**And** all 8 total nodes report `Ready` simultaneously
**And** no IP addresses are hardcoded in any playbook or inventory file (group names only)

### Story 1.4: Apply Control-Plane NoSchedule Taints

As an operator,
I want all 3 control-plane nodes tainted NoSchedule,
So that user workloads schedule exclusively on worker nodes.

**Acceptance Criteria:**

**Given** 3 control-plane + 5 worker nodes from Stories 1.1–1.3
**When** `ansible-playbook configure-nodes.yml` applies node taints
**Then** each control-plane node carries taint `node-role.kubernetes.io/control-plane:NoSchedule`
**And** a test `Pod` with no `tolerations` schedules only to worker nodes
**And** no ArgoCD or ESO manifest in the repository includes a `NoSchedule` toleration for control-plane nodes

---

## Epic 2: Cluster Foundation (ArgoCD + Namespaces + RBAC)

Install ArgoCD via Ansible Helm bootstrap; deploy the App-of-Apps root application; create all 6 namespaces with standard labels and RBAC baseline. ArgoCD takes over GitOps management of all subsequent components by the end of this epic.

### Story 2.1: Create Namespace Scaffolding with Standard Labels

As an operator,
I want all 6 cluster namespaces created via Ansible manifest apply,
So that component deployments have their target namespaces available before ArgoCD takes over.

**Acceptance Criteria:**

**Given** a tainted 8-node cluster from Epic 1
**When** `ansible-playbook bootstrap-namespaces.yml` runs
**Then** all 6 namespaces exist: `argocd`, `cert-manager`, `external-secrets`, `metallb-system`, `terminus-infra`, `terminus-platform`
**And** every namespace carries labels: `app.kubernetes.io/managed-by`, `app.kubernetes.io/part-of`, `terminus.io/domain`, `terminus.io/service`
**And** `kubectl get ns` confirms all 6 in `Active` phase
**And** all Ansible tasks in the playbook have a `name:` attribute

### Story 2.2: Create RBAC Baseline (ClusterRoles and ServiceAccounts)

As an operator,
I want ClusterRoles and ServiceAccounts created per the naming conventions,
So that components deployed later have the permissions they need from the start.

**Acceptance Criteria:**

**Given** all 6 namespaces from Story 2.1
**When** `ansible-playbook bootstrap-rbac.yml` applies RBAC manifests from `infra/k3s/manifests/rbac/`
**Then** all expected ClusterRoles and ServiceAccounts are present in the cluster
**And** all resources carry the 4 standard labels (`app.kubernetes.io/managed-by`, `app.kubernetes.io/part-of`, `terminus.io/domain`, `terminus.io/service`)
**And** no role grants `cluster-admin` to any non-operator ServiceAccount

### Story 2.3: Bootstrap ArgoCD via Ansible Helm Install

As an operator,
I want ArgoCD installed into the `argocd` namespace via Ansible running `helm install`,
So that the GitOps controller is running and able to manage subsequent component deployments.

**Acceptance Criteria:**

**Given** the `argocd` namespace from Story 2.1 and RBAC baseline from Story 2.2
**When** `ansible-playbook bootstrap-argocd.yml` runs `helm install argocd` with `infra/k3s/helm/argocd/values.yaml`
**Then** all ArgoCD pods are `Running` in the `argocd` namespace
**And** `helm ls -n argocd` shows the release in `deployed` state
**And** `infra/k3s/helm/argocd/values.yaml` is the sole source of Helm configuration (no `--set` flags used in the playbook)
**And** ArgoCD pods schedule only on worker nodes and carry no `NoSchedule` control-plane toleration

### Story 2.4: Deploy App-of-Apps Root Application

As an operator,
I want the ArgoCD App-of-Apps root application deployed,
So that ArgoCD manages all remaining cluster components via GitOps from this point forward.

**Acceptance Criteria:**

**Given** ArgoCD running from Story 2.3 and all ArgoCD `Application` manifests committed at `infra/k3s/argocd/apps/`
**When** `ansible-playbook deploy-root-app.yml` applies `infra/k3s/argocd/root-app.yaml`
**Then** the root `Application` resource is `Synced` and `Healthy` in ArgoCD
**And** child `Application` resources from `infra/k3s/argocd/apps/*.yaml` are visible in ArgoCD
**And** each child `Application` has `syncPolicy.automated.prune: true` and `selfHeal: true`
**And** `argocd app list` confirms the root app is managing downstream applications

---

## Epic 3: Network & Ingress (MetalLB + Traefik)

Deploy MetalLB in L2 mode with the confirmed IP pool, then configure Traefik to receive a LoadBalancer VIP from MetalLB and activate the `traefik` IngressClass. Both components managed via ArgoCD.

### Story 3.1: Deploy MetalLB with L2 IPAddressPool

As an operator,
I want MetalLB deployed in L2 mode with the `10.0.0.126–10.0.0.254` IP pool,
So that Kubernetes `LoadBalancer` services can receive routable IPs on the local network.

**Acceptance Criteria:**

**Given** ArgoCD App-of-Apps managing cluster from Epic 2 and the `metallb-system` namespace present
**When** the MetalLB `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/metallb.yaml`
**Then** all MetalLB pods (controller + speaker) are `Running` in `metallb-system`
**And** an `IPAddressPool` resource exists for `10.0.0.126–10.0.0.254`
**And** an `L2Advertisement` resource is present referencing that pool
**And** MetalLB Helm values are sourced exclusively from `infra/k3s/helm/metallb/values.yaml`
**And** all MetalLB resources carry the 4 standard labels

### Story 3.2: Configure Traefik with MetalLB VIP and Active IngressClass

As an operator,
I want Traefik deployed with its `LoadBalancer` service assigned a MetalLB VIP,
So that ingress routing is active and workloads can be exposed via `IngressClass: traefik`.

**Acceptance Criteria:**

**Given** MetalLB running and serving IPs from Story 3.1
**When** the Traefik `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/traefik.yaml`
**Then** Traefik pods are `Running` in the `terminus-infra` namespace
**And** the Traefik `Service` of type `LoadBalancer` has an external IP assigned from the `10.0.0.126–10.0.0.254` pool
**And** `kubectl get ingressclass` shows `traefik` as an available IngressClass
**And** a test `Ingress` resource using `ingressClassName: traefik` routes successfully to a test backend pod
**And** Traefik Helm values are sourced exclusively from `infra/k3s/helm/traefik/values.yaml`

---

## Epic 4: TLS & Secrets Infrastructure (cert-manager + ESO)

Register ESO's Vault Kubernetes auth pre-condition via Ansible, then deploy cert-manager (with Vault PKI ClusterIssuer) and ESO (with Vault SecretStore) via ArgoCD. No secrets committed to source control at any point.

### Story 4.1: Prepare ESO Vault Bootstrap Secret via Ansible

As an operator,
I want the ESO ServiceAccount JWT registered with the Vault Kubernetes auth backend via Ansible,
So that the ESO `SecretStore` can authenticate to Vault once ESO is deployed by ArgoCD.

**Acceptance Criteria:**

**Given** Vault running externally at `vault.trantor.internal` with Kubernetes auth backend enabled (pre-condition from `terminus-infra-secrets`)
**When** `ansible-playbook bootstrap-eso-secret.yml` runs
**Then** the ESO ServiceAccount JWT is registered with Vault's Kubernetes auth backend via `kubectl create secret` (not committed to source control)
**And** the secret exists in the `external-secrets` namespace at runtime
**And** no Vault token, JWT, or credential appears in any committed file
**And** the playbook is idempotent (safe to re-run)

### Story 4.2: Deploy cert-manager with Vault PKI ClusterIssuer

As an operator,
I want cert-manager deployed with a `ClusterIssuer` pointing at the Vault PKI internal CA,
So that workloads can request TLS certificates signed by the internal Vault CA.

**Acceptance Criteria:**

**Given** ArgoCD managing the cluster from Epic 2, `cert-manager` namespace present, and Vault reachable at `vault.trantor.internal`
**When** the cert-manager `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/cert-manager.yaml`
**Then** all cert-manager pods are `Running` in the `cert-manager` namespace
**And** a `ClusterIssuer` of type Vault PKI is present and reports `Ready: True`
**And** a test `Certificate` resource issued against the `ClusterIssuer` reaches `Ready: True` and contains a valid TLS cert signed by the Vault CA
**And** cert-manager Helm values are sourced exclusively from `infra/k3s/helm/cert-manager/values.yaml`
**And** all cert-manager resources carry the 4 standard labels

### Story 4.3: Deploy ESO with Vault SecretStore

As an operator,
I want External Secrets Operator deployed with a `SecretStore` configured for Vault Kubernetes auth,
So that workloads can retrieve secrets from Vault KV v2 without storing credentials in source control.

**Acceptance Criteria:**

**Given** the ESO bootstrap secret from Story 4.1 present in `external-secrets` namespace
**When** the ESO `Application` resource syncs via ArgoCD from `infra/k3s/argocd/apps/external-secrets.yaml`
**Then** all ESO pods are `Running` in the `external-secrets` namespace and carry no `NoSchedule` control-plane toleration
**And** a `ClusterSecretStore` (or `SecretStore`) is present in `external-secrets` and reports `Valid: True`
**And** a test `ExternalSecret` successfully syncs a known Vault KV v2 secret into a Kubernetes `Secret`
**And** ESO Helm values are sourced exclusively from `infra/k3s/helm/external-secrets/values.yaml`

---

## Epic 5: Consumer Contract Validation

Author and execute a consumer contract validation script covering all 8 testable contract items. The cluster is only considered "ready for handoff" when an operator has run the script and all items pass.

### Story 5.1: Author Consumer Contract Validation Script

As an operator,
I want a runnable validation script that checks all 8 consumer contract items,
So that "cluster ready" is a testable condition rather than a prose assertion.

**Acceptance Criteria:**

**Given** a fully deployed cluster from Epics 1–4
**When** `infra/k3s/scripts/validate-contract.sh` is authored and committed
**Then** the script checks all 8 contract items:
  1. All 8 nodes in `Ready` state
  2. All 6 namespaces in `Active` phase
  3. At least one `ExternalSecret` has `synced` status
  4. `ClusterIssuer` reports `Ready: True` and a test `Certificate` issues successfully
  5. MetalLB `IPAddressPool` is present and at least one IP has been allocated
  6. Traefik pods are `Running` and the `LoadBalancer` service has an external IP
  7. ArgoCD root app and all child apps are `Synced` and `Healthy`
  8. A default `StorageClass` is present and active
**And** each check produces a `PASS` or `FAIL` line with the item name
**And** the script exits non-zero if any check fails

### Story 5.2: Execute Contract Validation and Obtain Operator Sign-Off

As an operator,
I want to run the contract validation script against the deployed cluster and record the result,
So that there is a documented, dated sign-off confirming the cluster is ready for downstream consumers.

**Acceptance Criteria:**

**Given** the validation script from Story 5.1 committed and all Epics 1–4 complete
**When** an operator runs `infra/k3s/scripts/validate-contract.sh` against the live cluster
**Then** all 8 checks return `PASS`
**And** the output is captured and committed to `infra/k3s/docs/contract-validation-result.txt` with a datestamp
**And** the result file confirms zero `FAIL` lines

---

## Epic 6: Operational Lifecycle (Upgrade + Node Replace + Docs)

Author, test, and document the cluster upgrade and node replacement workflows. Bootstrap runbook authored. All 3 runbooks operator-validated before epic is considered complete.

### Story 6.1: Implement and Validate Cluster Upgrade Playbook and Runbook

As an operator,
I want an Ansible playbook and runbook for upgrading the k3s cluster version,
So that future upgrades are repeatable, documented, and do not require ad-hoc recovery.

**Acceptance Criteria:**

**Given** a running cluster from Epics 1–4
**When** `infra/k3s/ansible/upgrade.yml` is authored and a test upgrade run is performed (e.g., patch version bump)
**Then** the playbook drains each node in sequence, upgrades k3s, and uncordons without service interruption
**And** `kubectl get nodes` shows all nodes `Ready` on the new version after the run
**And** `infra/k3s/docs/runbook-upgrade.md` documents pre-conditions, step-by-step procedure, rollback steps, and expected output
**And** an operator has performed a dry-run or test upgrade and confirmed the runbook is accurate

### Story 6.2: Implement and Validate Node Replacement Playbook and Runbook

As an operator,
I want an Ansible playbook and runbook for replacing a failed worker or control-plane node,
So that node failure recovery is a documented, reproducible procedure.

**Acceptance Criteria:**

**Given** a running cluster from Epics 1–4
**When** `infra/k3s/ansible/node-replace.yml` is authored and validated against a simulated node removal
**Then** the playbook drains the target node, removes it from the cluster, and rejoins a replacement node
**And** `kubectl get nodes` confirms the replacement node reaches `Ready` state and workloads reschedule
**And** `infra/k3s/docs/runbook-node-replace.md` documents drain, remove, rejoin steps, and expected output
**And** an operator has validated the runbook accuracy against an actual or simulated node replacement

### Story 6.3: Author Bootstrap Runbook

As an operator,
I want a bootstrap runbook that documents the full cluster initialization procedure from scratch,
So that any operator can reproduce the cluster without tribal knowledge.

**Acceptance Criteria:**

**Given** all Ansible playbooks and ArgoCD manifests committed from Epics 1–5
**When** `infra/k3s/docs/runbook-bootstrap.md` is authored
**Then** the runbook covers: pre-requisites (Vault pre-conditions, inventory setup), Step 1–10 delivery sequence, expected output at each step, and validation via the contract script
**And** the runbook references all relevant playbooks and their options
**And** an operator unfamiliar with the implementation has reviewed the runbook and confirmed it is complete and accurate
**And** the runbook is committed to the repository alongside the other operational docs
