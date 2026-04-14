---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments:
  - _bmad-output/lens-work/initiatives/terminus/infra/phases/preplan/brainstorm-notes.md
date: "2026-03-21"
author: Todd
ported_from: ""
scope_changes:
  - "New feature split: Proxmox is now a dedicated infra feature rather than implicit substrate work"
  - "Scope explicitly ends at virtualization and VM provisioning boundaries"
  - "k3s, secrets, postgres, and temporal are treated as downstream consumers, not responsibilities of this feature"
---

# Product Brief: Proxmox Infrastructure Substrate

---

## Executive Summary

The Proxmox feature (`terminus-infra-proxmox`) establishes the virtualization substrate for the Terminus environment. It defines how foundational infrastructure VMs are provisioned, templated, networked, secured, and recovered inside Proxmox so that higher-layer infrastructure can be deployed repeatably. This feature owns the hypervisor-facing automation and operational model required to produce reliable VM capacity for Terminus.

This feature does not own Kubernetes, application deployment, secrets, databases, or Temporal. Instead, it creates the stable base layer on which those capabilities can be installed. The central problem is infrastructure drift: without a clear Proxmox substrate model, every higher-layer feature becomes fragile, harder to recover, and more dependent on undocumented operator memory.

The core insight: Proxmox is not just where VMs happen to run. It is an explicit productized substrate whose interfaces must be stable enough that downstream features can depend on it confidently.

---

## Core Vision

### Problem Statement

Terminus needs a reliable and repeatable infrastructure substrate before cluster and platform capabilities can be deployed. Hand-created virtual machines, ad hoc network attachment, undocumented templates, and unplanned backup/recovery behavior create a brittle environment where every downstream feature inherits operational risk.

### Problem Impact

- Rebuilds are slow and error-prone when VM creation depends on manual steps
- Network assumptions drift between infrastructure components
- Backups and recovery become guesswork during failures
- Security posture weakens when templates or provisioning processes are informal
- Downstream work on k3s and platform capabilities is blocked or destabilized

### Why Existing Approaches Fall Short

- **Manual VM creation:** Fast initially, but not repeatable and not reviewable
- **Snowflake templates:** Hard to audit, hard to update consistently
- **Implicit network design:** Causes hidden constraints that surface later during cluster or ingress work
- **Backup-later thinking:** Creates unacceptable recovery risk for foundational infrastructure

### Proposed Solution

A Proxmox-focused infrastructure feature that:

1. Defines standard VM roles for the Terminus environment
2. Establishes reproducible template and provisioning patterns
3. Defines network attachment and addressing expectations for infra VMs
4. Establishes snapshot, backup, and restore expectations at the hypervisor layer
5. Documents the contract handed to downstream consumers such as `terminus-infra-k3s`

### Key Differentiators

- **Substrate-first design:** Treats virtualization as a productized foundation, not a side effect
- **Reproducibility over convenience:** VM templates and provisioning workflows are expected to be repeatable
- **Explicit ownership boundary:** Clear separation from k3s, secrets, postgres, and temporal features
- **Recovery-aware:** Backup and rebuild planning are part of the MVP, not postponed work

---

## Target Users

### Primary Users

**Todd — Homelab Operator and Infrastructure Owner**

- Context: Maintains the Terminus homelab and is responsible for its recoverability
- Challenge: Needs a repeatable way to provision and maintain the VM substrate without accumulating hidden infrastructure debt
- Motivation: Build a foundation that lets higher-layer services be deployed safely and rebuilt predictably
- Success moment: Can provision or rebuild the VM substrate from documented artifacts with minimal improvisation

### User Journey

1. Operator reviews the documented VM roles and substrate architecture
2. Operator provisions or updates Proxmox-backed VMs using the approved workflow
3. Infrastructure nodes appear with expected naming, network attachment, and base hardening
4. Backups and snapshots are configured per the defined policy
5. Downstream features consume the resulting VMs as stable infrastructure inputs

---

## Success Metrics

| Metric | Target |
|---|---|
| VM provisioning repeatability | 100% of standard VM roles can be recreated from documented workflow |
| Manual post-provision fixes | 0 required for standard VM classes |
| Backup coverage | 100% of substrate VMs included in documented backup policy |
| Recovery confidence | Operator can rebuild a failed standard VM from docs and artifacts within 1 hour |
| Security hygiene | 0 secrets or credentials committed to source control |

---

## MVP Scope — This Feature

### Owned by terminus-infra-proxmox

**1. Proxmox VM Substrate Design**
- Standard VM role definitions for Terminus infrastructure
- Naming and host-role conventions
- Resource sizing baselines for standard VM classes

**2. Provisioning & Templates**
- Repeatable provisioning workflow for standard infrastructure VMs
- Base image/template expectations and hardening requirements
- Documented handoff state for downstream infra consumers

**3. Network Foundations**
- Virtual network attachment strategy for infra VMs
- Addressing/reservation conventions where needed
- Clear access expectations for operators and downstream services

**4. Backup & Recovery Baseline**
- Snapshot policy for risky changes
- Backup expectations and retention baseline
- Restore/rebuild runbook assumptions for VM failure scenarios

### Out of Scope — Handled by Other Features

| Responsibility | Feature |
|---|---|
| k3s cluster installation and lifecycle | `terminus-infra-k3s` |
| Secret manager and secret distribution | `terminus-infra-secrets` |
| Postgres deployment and database operations | `terminus-infra-postgres` |
| Crossplane-managed shared services | `terminus-infra-crossplane` |
| Temporal orchestration platform | `terminus-platform-temporal` |

### Out of Scope — Future

- Advanced HA/multi-host evacuation automation beyond MVP needs
- Non-Terminus tenant support
- Full bare-metal lifecycle management outside Proxmox scope

---

## Architecture Notes (MVP)

**Stack direction:** Proxmox-native provisioning and operator automation, with final implementation tooling to be chosen in later phases.

**Boundary:**
```
Proxmox substrate
  ├── templates / VM roles / network attachment
  ├── provisioning workflow
  ├── backup + recovery baseline
  └── handoff to terminus-infra-k3s
```

**Resilience principle:** The substrate must support rebuild and recovery without undocumented operator steps.

**Security principle:** No passwords, API tokens, or private keys in source control. Provisioning artifacts must assume external secret injection and hardened base images.
