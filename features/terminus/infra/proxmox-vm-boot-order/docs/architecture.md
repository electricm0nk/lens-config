---
feature: proxmox-vm-boot-order
doc_type: architecture
status: draft
goal: "Add deterministic VM boot ordering to the Proxmox cluster so the system recovers gracefully after a hypervisor power cycle or host restart"
key_decisions:
  - Use bpg/proxmox native startup{} blocks as the single mechanism for boot order control
  - Assign tiers with order= values; use up_delay= for gate delay between tiers
  - Bring automation-dev (VM 9001) under a new tofu/modules/automation-host module
  - Workers get distinct sequential order= values so each starts individually
open_questions:
  - up_delay values are initial estimates; calibrate against observed guest-agent response times during acceptance
assumptions:
  - All VMs in the boot sequence reside on the same Proxmox physical node (trantor); cross-node boot ordering is not supported by Proxmox autostart
depends_on: []
blocks: []
updated_at: "2026-04-17"
---

# Architecture: Proxmox VM Boot Order Control

**Feature:** `proxmox-vm-boot-order`  
**Track:** tech-change  
**Domain/Service:** terminus / infra  

---

## Problem Statement

The Proxmox cluster currently has no configured VM autostart ordering. When the hypervisor recovers from a power cycle or reboot, all VMs with autostart enabled are sent their start signal simultaneously. This causes race conditions:

- k3s control-plane nodes attempt to join the cluster before their Consul backend is reachable
- k3s workers begin registering workloads before the control-plane quorum is established
- PostgreSQL (Patroni) attempts DCS leader election against a Consul agent that has not yet fully started
- The automation host (VM 9001) — which runs OpenTofu, Vault CLI, SOPS, and Ansible — may not be available when other services need it for initialization

The result is a non-deterministic recovery sequence that requires manual operator intervention to stabilize after every unplanned restart.

---

## Solution Overview

The `bpg/proxmox` provider exposes a `startup` block on `proxmox_virtual_environment_vm` resources with three fields:

| Field | Effect |
|-------|--------|
| `order` | Integer priority; lower values start first. Proxmox processes autostart VMs in ascending order. |
| `up_delay` | Seconds to wait **after** this VM's QEMU guest agent reports running before Proxmox moves on to the next order tier. Requires `agent { enabled = true }`. |
| `down_delay` | Seconds to wait before sending the shutdown signal to the next VM in descending order. |

All existing `proxmox_virtual_environment_vm` resources already have `agent { enabled = true }`, so `up_delay` gates operate against a real liveness signal — not a timer from power-on.

### Why startup{} over Proxmox datacenter.cfg

Proxmox also supports `startuporder` via `qm set` or GUI. That approach is:
- Not tracked in version control
- Not reproducible from `tofu apply`
- Drift-prone — `tofu plan` would show no diff even if ordering was manually changed

Using native `startup {}` blocks keeps boot ordering declared alongside every other VM property, is visible in `tofu plan` output, and is enforced on every apply.

---

## Boot Sequence Tiers

```
Tier 1 ──── automation-dev (VM 9001)
              └─ gate: 120s after guest agent responds

Tier 2 ──── postgres-primary (VM 200)
              └─ gate: 90s after guest agent responds

Tier 3a ─── postgres-replica (VM 201)   ─┐  concurrent within tier
Tier 3b ─── postgres-backup (VM 202)    ─┘
              └─ gate: 60s after last guest agent in tier responds

Tier 4a ─── k3s-cp-01 (VM 300)
Tier 4b ─── k3s-cp-02 (VM 301)
Tier 4c ─── k3s-cp-03 (VM 302)
              └─ gate: 60s after each cp VM's guest agent responds

Tier 5a ─── k3s-worker-01 (VM 310)     ─┐  sequential, one at a time
Tier 5b ─── k3s-worker-02 (VM 311)      │
Tier 5c ─── k3s-worker-03 (VM 312)      │
Tier 5d ─── k3s-worker-04 (VM 313)      │
Tier 5e ─── k3s-worker-05 (VM 314)     ─┘
              └─ gate: 60s between each worker
```

**Order value mapping:**

| VM | Role | order | up_delay |
|----|------|-------|----------|
| 9001 | automation-dev | 1 | 120 |
| 200 | postgres-primary | 2 | 90 |
| 201 | postgres-replica | 3 | 60 |
| 202 | postgres-backup | 3 | 60 |
| 300 | k3s-cp-01 | 4 | 60 |
| 301 | k3s-cp-02 | 5 | 60 |
| 302 | k3s-cp-03 | 6 | 60 |
| 310 | k3s-worker-01 | 7 | 60 |
| 311 | k3s-worker-02 | 8 | 60 |
| 312 | k3s-worker-03 | 9 | 60 |
| 313 | k3s-worker-04 | 10 | 60 |
| 314 | k3s-worker-05 | 11 | 60 |

**Tier 3 concurrency:** postgres-replica and postgres-backup share `order = 3`. Proxmox will start both simultaneously within the tier. Both are non-primary and have no ordering dependency between themselves. The `up_delay` applies per-VM and the tier does not advance until all order=3 VMs have completed their gate.

**Worker sequencing:** Each worker gets a unique `order` value (7–11). Combined with `up_delay = 60`, each worker has 60 seconds of readiness time before the next one starts. This ensures node registration does not flood the control-plane simultaneously.

---

## Component Changes

### 1. New module: `tofu/modules/automation-host`

The automation-dev VM (VM 9001) is currently not under OpenTofu management. This feature introduces a new module to declare it as a managed resource with boot ordering.

**File:** `tofu/modules/automation-host/main.tf`

```hcl
resource "proxmox_virtual_environment_vm" "automation_host" {
  name      = var.vm_name
  node_name = var.target_node
  vm_id     = var.vm_id  # 9001

  on_boot = true

  startup {
    order      = 1
    up_delay   = 120
    down_delay = 60
  }

  agent {
    enabled = true
    timeout = "15m"
    trim    = false
  }

  # ... remaining VM configuration (cpu, memory, disk, network, cloud-init)
}
```

**Environment integration:** A new environment root `tofu/environments/automation/dev/` will compose this module for the dev environment, following the same pattern as `tofu/environments/k3s/dev/`.

### 2. `tofu/modules/postgres-cluster`

Add `startup {}` blocks to both `postgres_primary` and `postgres_replica` resources.

**`proxmox_virtual_environment_vm.postgres_primary`:**
```hcl
on_boot = true

startup {
  order      = 2
  up_delay   = 90
  down_delay = 60
}
```

**`proxmox_virtual_environment_vm.postgres_replica`:**
```hcl
on_boot = true

startup {
  order      = 3
  up_delay   = 60
  down_delay = 30
}
```

### 3. `tofu/modules/postgres-backup`

Add `startup {}` block to `proxmox_virtual_environment_vm.postgres_backup`:

```hcl
on_boot = true

startup {
  order      = 3
  up_delay   = 60
  down_delay = 30
}
```

### 4. `tofu/modules/k3s-cluster`

The k3s module uses `for_each` across `local.all_nodes`. Boot ordering requires each VM to have a distinct `order` value. This requires augmenting the local node map to carry a `startup_order` field derived from the VM's index.

**Updated locals:**

```hcl
locals {
  control_plane_nodes = {
    for index, hostname in local.control_plane_names : hostname => {
      vm_id         = var.control_plane_vm_id_base + index
      role          = "control-plane"
      cpu           = var.control_plane_cpu
      memory_mb     = var.control_plane_memory_mb
      startup_order = var.control_plane_startup_order_base + index
    }
  }

  worker_nodes = {
    for index, hostname in local.worker_names : hostname => {
      vm_id         = var.worker_vm_id_base + index
      role          = "worker"
      cpu           = var.worker_cpu
      memory_mb     = var.worker_memory_mb
      startup_order = var.worker_startup_order_base + index
    }
  }
}
```

**New variables:**

```hcl
variable "control_plane_startup_order_base" {
  description = "Starting Proxmox autostart order value for control-plane nodes. Each CP gets base+index."
  type        = number
  default     = 4
}

variable "worker_startup_order_base" {
  description = "Starting Proxmox autostart order value for worker nodes. Each worker gets base+index."
  type        = number
  default     = 7
}

variable "startup_up_delay" {
  description = "Seconds to wait after QEMU agent responds before starting the next-order VM."
  type        = number
  default     = 60
}

variable "startup_down_delay" {
  description = "Seconds to wait before sending shutdown signal to next VM during Proxmox autostart shutdown."
  type        = number
  default     = 30
}
```

**Updated resource:**

```hcl
resource "proxmox_virtual_environment_vm" "nodes" {
  for_each = local.all_nodes

  # ... existing config unchanged ...

  on_boot = true

  startup {
    order      = each.value.startup_order
    up_delay   = var.startup_up_delay
    down_delay = var.startup_down_delay
  }
}
```

**Environment variable overrides** (`tofu/environments/k3s/dev/main.tf`):

```hcl
module "k3s_cluster" {
  # ... existing vars ...
  control_plane_startup_order_base = 4
  worker_startup_order_base        = 7
  startup_up_delay                 = 60
  startup_down_delay               = 30
}
```

---

## Proxmox Autostart Prerequisite

The `startup {}` block only takes effect when **Proxmox autostart** is enabled on the VM (`onboot = true` in `bpg/proxmox`). All modules must set this explicitly.

Add to each `proxmox_virtual_environment_vm` resource:

```hcl
on_boot = true
```

This field is already supported by the `bpg/proxmox` provider (reference: provider attribute `on_boot`). Without it, the `startup {}` block has no effect — Proxmox does not autostart the VM at all.

---

## ADR-001: startup{} Blocks as Sole Boot Order Mechanism

- **Status:** Accepted
- **Decision:** All boot ordering is declared via `startup {}` blocks in `proxmox_virtual_environment_vm` resources managed by OpenTofu. No ordering is configured via `qm set`, the Proxmox GUI, or `datacenter.cfg`.
- **Rationale:** Single source of truth. Drift is visible in `tofu plan`. Reproducible from `tofu apply` on any environment.
- **Consequences:** All VMs must be managed resources in OpenTofu to participate in boot ordering.

## ADR-002: Bring automation-dev Under OpenTofu Management

- **Status:** Accepted
- **Decision:** A new `tofu/modules/automation-host` module is introduced to declare VM 9001 as a managed resource. An `import {}` block is declared in the module to absorb the existing VM into state on first apply.
- **Rationale:** Boot ordering via `startup {}` only works on `proxmox_virtual_environment_vm` resources managed by OpenTofu state. An unmanaged VM cannot be given an OpenTofu-declared `startup {}` block. Using an `import {}` block (rather than a manual `tofu import` pre-step) keeps the import intent in version control — any operator can run `tofu apply` safely without a documented pre-step.
- **Consequences:** The module's declared configuration (CPU, memory, disk) must match the existing VM's actual configuration on first apply. The operator must run `tofu plan` in review mode before the first apply to confirm no destructive changes are planned. The `import {}` block should be retained in the module; OpenTofu is idempotent once the resource is already in state. Boot order acceptance testing requires a controlled hypervisor reboot — this cannot be verified via `tofu plan` alone. See [BOOT-ORDER-ACCEPTANCE] in `.todo`.

## ADR-003: Shared order= Values for Concurrent Tier Members

- **Status:** Accepted
- **Decision:** postgres-replica and postgres-backup share `order = 3` to start concurrently within their tier.
- **Rationale:** Neither depends on the other at boot time. Forcing sequential start of replica+backup adds unnecessary recovery delay with no operational benefit.
- **Consequences:** `up_delay` applies independently per-VM; Proxmox does not advance to the next tier until all order=3 VMs have completed their up_delay gate.

---

## Assumptions

1. **Single Proxmox physical node:** All VMs in the boot sequence reside on the same Proxmox node (`trantor`). Proxmox autostart ordering (`startup.order`) is enforced per-hypervisor-node. If VMs are distributed across multiple physical nodes, the ordering guarantee does not apply across nodes and this architecture must be revisited.
2. **QEMU guest agent is running:** All `proxmox_virtual_environment_vm` resources already have `agent { enabled = true }`. The `up_delay` gate fires against the guest agent's liveness signal, not a wall-clock timer from power-on. If the guest agent fails to start, Proxmox falls back to a fixed timeout before advancing to the next order tier.
3. **Recovery time is not SLO-bounded:** The fully-sequenced boot takes approximately 12–15 minutes from hypervisor power-on to all workers running. This is the accepted tradeoff. Out-of-order startup causes cascading job recycling that takes hours to stabilize; the sequenced delay is the correct tradeoff.
4. **up_delay values are initial estimates:** The values (120s, 90s, 60s) are conservative starting points, not calibrated measurements. They should be validated by observation during acceptance testing and tuned after the first controlled reboot. See open_questions.

## Out of Scope

- Proxmox cluster HA fencing or migration ordering — this feature covers single-node autostart sequencing only
- Application-level health checks (k3s node-ready, Patroni leader election) — those are Ansible/k3s concerns; `up_delay` provides a time buffer but does not perform application-layer readiness polling
- Shutdown ordering (`down_delay`) — values are set conservatively for safe shutdown, but optimizing shutdown sequence is not a goal of this feature
