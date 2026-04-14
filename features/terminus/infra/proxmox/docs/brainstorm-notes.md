---
stepsCompleted: [1]
session_topic: "Proxmox substrate for Terminus infrastructure"
session_goals:
  - Define the Proxmox-hosted compute foundation required for Terminus
  - Establish VM topology and host-role strategy for downstream k3s workloads
  - Define network segmentation, addressing, and access expectations
  - Clarify storage, backup, snapshot, and recovery expectations at the Proxmox layer
  - Separate Proxmox responsibilities from k3s, secrets, postgres, and temporal ownership
selected_approach: ""
techniques_used: []
ideas_generated: 0
context_file: ""
date_created: "2026-03-21"
ported_from: ""
scope_note: "New planning cycle. This feature owns the Proxmox virtualization substrate for Terminus. It provides the VM and network foundation on which k3s and all higher-layer infrastructure will run."
---

# Proxmox Infrastructure Brainstorming Session

## Session Overview

**Topic:** Proxmox substrate for Terminus infrastructure
Defines the virtualization layer for the Terminus homelab environment. This feature owns how Terminus compute is represented in Proxmox: node templates, VM classes, storage placement, backup expectations, and network attachment patterns for downstream infrastructure.

**Core Constraint:** Proxmox is foundational infrastructure. Errors here propagate upward into k3s, secrets, postgres, temporal, and every feature workload. The design must favor recoverability and repeatability over convenience.

### Session Goals

✓ Define the role of Proxmox in the Terminus stack
✓ Identify the VM classes and network expectations needed by downstream services
✓ Establish storage, backup, and snapshot expectations at the hypervisor layer
✓ Clarify security boundaries at the infrastructure substrate
✓ Separate Proxmox responsibilities from higher-layer cluster and application concerns

---

## Morphological Analysis Results

### Parameter 1: Infrastructure Role

**Accepted role for this feature:**
- Proxmox is the virtualization substrate for Terminus
- It provides VMs, templates, placement rules, virtual networks, and backup primitives
- It does **not** own Kubernetes, application deployment, or service configuration

**Implication:** This feature must stop at "the VM is ready for a consumer such as k3s".

### Parameter 2: VM Topology

**Likely VM classes required:**
- **Control-plane VMs** for the future k3s control plane
- **Worker VMs** for future workload execution
- **Support VMs** for supporting infrastructure where warranted
- **Template images** used to stamp out the above reproducibly

**Design preference:** Small number of clear VM roles, not many one-off handcrafted VMs.

### Parameter 3: Network Model

**Required network decisions:**
- Management access path to Proxmox and VMs
- Cluster/internal traffic separation where practical
- Stable addressing or reservation strategy for infra VMs
- Clear ingress path for downstream services without conflating it with Proxmox ownership

**Constraint:** Network design must support later k3s ingress, service discovery, and secure admin access without forcing a redesign.

### Parameter 4: Storage & Recovery

**Required substrate guarantees:**
- VM disk placement strategy
- Snapshot policy for safe changes
- Backup schedule and retention expectations
- Restore/rebuild procedure for failed VM or host scenarios

**Bias:** Prefer repeatable rebuilds plus backups over snowflake VM recovery.

### Parameter 5: Security Boundaries

**Security requirements at the Proxmox layer:**
- Least-privilege admin access
- No secrets committed into VM templates or provisioning artifacts
- Hardened base images
- Auditability of VM creation and network attachment decisions

**Constraint from org constitution:** Security-first applies here. Proxmox automation must not embed passwords, tokens, or private keys in source control.

### Parameter 6: Downstream Contracts

**This feature should provide to downstream consumers:**
- Documented VM classes and intended use
- Provisioning workflow for reproducible VM creation
- Naming conventions and address conventions
- Backup and recovery expectations
- Clear handoff point to `terminus-infra-k3s`

**This feature should not provide:**
- k3s installation
- secrets management
- postgres deployment
- temporal deployment
- application namespaces or Helm releases

---

## Design Decisions Carried Forward

- **Proxmox owns substrate, not workloads:** This feature ends at the virtualization boundary
- **Templates over handcrafted VMs:** Reproducibility is mandatory
- **Backups and rebuilds are first-class:** Recovery must be planned, not improvised
- **Network and access design must anticipate k3s:** The substrate should not block cluster growth
- **Security-first at the hypervisor layer:** No credential leakage into templates or provisioning code

---

## Out of Scope for This Feature

- k3s cluster installation and lifecycle → `terminus-infra-k3s`
- Secret manager deployment and secret distribution → `terminus-infra-secrets`
- Postgres service deployment and operations → `terminus-infra-postgres`
- Temporal service deployment and scheduling → `terminus-platform-temporal`
- Crossplane configuration of shared infra services → `terminus-infra-crossplane`
