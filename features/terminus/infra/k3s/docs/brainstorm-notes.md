---
stepsCompleted: [1]
session_topic: "k3s cluster substrate for Terminus workloads"
session_goals:
  - Define the k3s cluster role within the Terminus stack
  - Establish control-plane and worker topology expectations
  - Clarify ingress, storage, namespace, and baseline operator needs
  - Define the handoff boundaries between k3s and secrets, postgres, and temporal features
  - Establish security and recovery expectations for the cluster layer
selected_approach: ""
techniques_used: []
ideas_generated: 0
context_file: ""
date_created: "2026-03-21"
ported_from: ""
scope_note: "New planning cycle. This feature owns the Kubernetes cluster substrate for Terminus. It consumes the Proxmox VM foundation and provides the runtime environment on which higher-layer infra and platform features will run."
---

# k3s Infrastructure Brainstorming Session

## Session Overview

**Topic:** k3s cluster substrate for Terminus workloads
Defines the Kubernetes cluster layer for the Terminus environment. This feature owns how k3s is installed, structured, secured, and operated as the shared runtime for Terminus services.

**Core Constraint:** k3s is the boundary where virtualization becomes a shared application runtime. If the cluster model is unstable, every higher-layer feature inherits avoidable deployment and operational complexity.

### Session Goals

✓ Define the role of k3s in the Terminus stack
✓ Identify cluster topology and baseline platform expectations
✓ Clarify ingress, storage, namespace, and policy foundations
✓ Separate k3s ownership from secrets, postgres, and temporal feature ownership
✓ Establish upgrade and recovery expectations for the cluster layer

---

## Morphological Analysis Results

### Parameter 1: Cluster Role

**Accepted role for this feature:**
- k3s is the shared runtime substrate for Terminus workloads
- It provides cluster formation, node joining, namespace foundations, ingress integration, and baseline operational configuration
- It does **not** own application-specific deployments or shared service implementations beyond the cluster substrate itself

### Parameter 2: Cluster Topology

**Required decisions:**
- Single-node vs. multi-node MVP stance
- Control-plane placement and quorum expectations
- Worker node role strategy
- Upgrade-safe node replacement approach

**Bias:** Prefer a topology that supports growth without making the MVP operationally fragile.

### Parameter 3: Baseline Cluster Capabilities

**Expected cluster foundations:**
- Ingress path for downstream services
- Storage class strategy for stateful workloads
- Namespace conventions for service separation
- RBAC and service account expectations
- Base observability/operability hooks sufficient for debugging and recovery

### Parameter 4: Security & Policy

**Required cluster-level security posture:**
- Hardened cluster bootstrap
- Administrative access boundaries
- Separation between cluster operations and application secrets
- No secrets committed in manifests or bootstrap code
- Baseline policy model that does not require each feature to invent its own foundation

### Parameter 5: Recovery & Lifecycle

**Required lifecycle decisions:**
- Cluster bootstrap workflow
- Node replacement workflow
- Upgrade procedure
- Backup or restoration expectations for cluster state where applicable

**Constraint:** Recovery must be documented early. A cluster that can only be maintained from memory is not acceptable.

### Parameter 6: Downstream Contracts

**This feature should provide to downstream consumers:**
- Stable cluster runtime for infra and platform workloads
- Documented namespace and ingress expectations
- Storage and service exposure baseline
- Handoff point for secrets, postgres, and temporal features

**This feature should not provide:**
- Secret manager implementation
- Postgres implementation
- Temporal implementation
- Application-specific Helm releases or worker deployments

---

## Design Decisions Carried Forward

- **k3s is a runtime substrate, not a catch-all feature:** It stops at the shared cluster layer
- **Baseline capabilities must be explicit:** ingress, storage, namespaces, and access patterns are part of the product
- **Security-first at the cluster layer:** no plaintext secrets in bootstrap manifests or cluster code
- **Operational recoverability matters:** bootstrap, upgrade, and node replacement must be documented from the start
- **Downstream ownership stays downstream:** secrets, postgres, and temporal remain separate features

---

## Out of Scope for This Feature

- Proxmox VM substrate and virtualization layer → `terminus-infra-proxmox`
- Secret manager deployment and secret distribution → `terminus-infra-secrets`
- Postgres deployment and database operations → `terminus-infra-postgres`
- Crossplane-managed shared services beyond cluster substrate → `terminus-infra-crossplane`
- Temporal orchestration platform → `terminus-platform-temporal`
