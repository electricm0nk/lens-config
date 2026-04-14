---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments:
  - _bmad-output/lens-work/initiatives/terminus/infra/phases/preplan/brainstorm-notes.md
date: "2026-03-21"
author: Todd
ported_from: ""
scope_changes:
  - "New feature split: k3s is now a dedicated infra feature rather than implicit platform setup work"
  - "Scope explicitly ends at cluster substrate and baseline operational capabilities"
  - "Secrets, postgres, crossplane-managed shared services, and temporal are treated as downstream consumers"
---

# Product Brief: k3s Cluster Substrate

---

## Executive Summary

The k3s feature (`terminus-infra-k3s`) establishes the Kubernetes cluster substrate for the Terminus environment. It defines how the cluster is bootstrapped, structured, secured, and operated so that infra-owned shared services and platform-owned orchestration components can run on a stable, repeatable runtime.

This feature consumes the Proxmox VM foundation and produces a working cluster baseline with clear ingress, namespace, storage, and access expectations. It does not own shared service implementations like secrets, postgres, or temporal. Its purpose is to make those features deployable without each one having to invent the cluster foundation independently.

The core insight: a Kubernetes cluster is not just an install step. It is a long-lived operational product with its own boundaries, upgrade story, security posture, and consumer contracts.

---

## Core Vision

### Problem Statement

Terminus needs a repeatable and supportable Kubernetes runtime before higher-layer infrastructure and platform services can be deployed safely. Ad hoc cluster creation, unclear namespace boundaries, missing ingress/storage conventions, and undocumented upgrade/recovery workflows create cascading instability for every downstream feature.

### Problem Impact

- Higher-layer services cannot rely on stable cluster assumptions
- Storage and ingress decisions become inconsistent across services
- Recovery and upgrade operations become risky and undocumented
- Security posture drifts when cluster foundations are assembled informally
- Platform work is blocked by unresolved runtime concerns

### Why Existing Approaches Fall Short

- **One-off installs:** Fast to start, but hard to reproduce and review
- **Application-first cluster design:** Forces each downstream service to solve the same runtime problems repeatedly
- **Implicit namespace/storage decisions:** Leads to coupling and inconsistent operations
- **Late recovery planning:** Turns routine upgrades and failures into outages

### Proposed Solution

A k3s-focused infrastructure feature that:

1. Defines the target cluster topology and bootstrap process
2. Establishes ingress, storage, namespace, and access conventions
3. Defines baseline operational practices for upgrades, node replacement, and recovery
4. Produces a stable runtime contract consumed by secrets, postgres, and temporal features

### Key Differentiators

- **Cluster-as-product mindset:** Treats the shared runtime as a governed capability, not a one-time install
- **Explicit consumer contract:** Downstream infra and platform features consume a documented cluster baseline
- **Security-first foundation:** Cluster setup avoids insecure shortcuts and secret sprawl
- **Operationally grounded:** Upgrade and recovery considerations are part of MVP planning

---

## Target Users

### Primary Users

**Todd — Homelab Operator and Platform Builder**

- Context: Needs a stable cluster runtime for all higher-layer Terminus services
- Challenge: Wants Kubernetes flexibility without inheriting unnecessary operational chaos
- Motivation: Build a cluster that can host future services predictably and be maintained without tribal knowledge
- Success moment: Can bootstrap and operate the cluster from documented artifacts and use it as a dependable substrate for new features

### User Journey

1. Operator provisions the required VMs from the Proxmox substrate
2. Operator bootstraps the k3s cluster using the documented workflow
3. Cluster baseline capabilities are established: ingress, namespaces, storage expectations, and admin access patterns
4. Downstream features deploy into the cluster using the documented contract
5. Operator performs an upgrade or node replacement without improvisation

---

## Success Metrics

| Metric | Target |
|---|---|
| Cluster bootstrap repeatability | 100% reproducible from documented workflow |
| Namespace and ingress consistency | 100% of downstream services follow documented cluster conventions |
| Upgrade confidence | Operator can execute the documented upgrade workflow without unplanned manual repair |
| Recovery confidence | Node replacement or cluster rebuild workflow documented and validated |
| Security hygiene | 0 secrets or credentials committed to source control |

---

## MVP Scope — This Feature

### Owned by terminus-infra-k3s

**1. Cluster Bootstrap & Topology**
- Target node topology for the Terminus cluster
- Cluster bootstrap workflow and base node join process
- Control-plane and worker role expectations

**2. Shared Runtime Foundations**
- Namespace model for service separation
- Ingress baseline for downstream service exposure
- Storage class and persistent volume expectations for stateful workloads
- Access and RBAC baseline at the cluster level

**3. Operational Baseline**
- Cluster upgrade expectations
- Node replacement workflow
- Base troubleshooting and recovery assumptions

**4. Downstream Consumer Contract**
- Clear documented handoff to `terminus-infra-secrets`, `terminus-infra-postgres`, and `terminus-platform-temporal`
- Documented expectations for how cluster consumers deploy and operate

### Out of Scope — Handled by Other Features

| Responsibility | Feature |
|---|---|
| Proxmox virtualization substrate | `terminus-infra-proxmox` |
| Secret manager and secret distribution | `terminus-infra-secrets` |
| Postgres deployment and database operations | `terminus-infra-postgres` |
| Crossplane-managed shared services | `terminus-infra-crossplane` |
| Temporal orchestration platform | `terminus-platform-temporal` |

### Out of Scope — Future

- Multi-cluster federation
- Advanced policy engines beyond MVP operational needs
- Full observability platform beyond baseline cluster operability

---

## Architecture Notes (MVP)

**Stack direction:** k3s as the Kubernetes distribution, consuming infrastructure VMs supplied by `terminus-infra-proxmox`. Final implementation tooling to be selected in later phases.

**Boundary:**
```
k3s cluster substrate
  ├── bootstrap + node join
  ├── ingress + namespace + storage baseline
  ├── operational lifecycle (upgrade / replace / recover)
  └── handoff to secrets, postgres, and temporal features
```

**Resilience principle:** Cluster operations must be repeatable and recoverable from documented procedures.

**Security principle:** Cluster bootstrap and management must not depend on secrets stored in source control. Baseline access controls and administrative boundaries must be explicit.
