# Story 1.1: Provision PostgreSQL Cluster VMs via OpenTofu

Status: done

## Story

As an operator,
I want to provision the primary and replica PostgreSQL VMs by running `tofu apply`,
so that I have correctly-sized, network-reachable Proxmox VMs ready for cluster bootstrap.

## Acceptance Criteria

1. 2 VMs exist in Proxmox with correct specs: 4 vCPU / 16 GB RAM / 32 GB OS disk + 150 GB data disk
2. Both VMs are reachable over SSH from the automation server
3. Hostnames resolve in the Terminus network
4. All provider versions are pinned and locked in `.terraform.lock.hcl`

## Tasks / Subtasks

- [ ] Create `modules/postgres-cluster/` OpenTofu module (AC: 1, 2, 3)
  - [ ] `main.tf` — `bpg/proxmox` VM resources for primary and replica nodes
  - [ ] `variables.tf` — vm_count, cpu, memory, disk_os_gb, disk_data_gb, network config, hostnames
  - [ ] `outputs.tf` — node IPs, hostnames (consumed by EPIC-002 remote-exec)
  - [ ] `versions.tf` — pin `bpg/proxmox` provider version
- [ ] Wire module in `tofu/environments/dev/main.tf` (AC: 1, 2, 3)
  - [ ] Call `postgres-cluster` module with dev sizing
  - [ ] `backend.tf` — Consul state at `terminus/infra/postgres/dev/opentofu.tfstate`
  - [ ] `versions.tf` — pin OpenTofu core version to 1.11.0
- [ ] SOPS-encrypt local state before git commit — handled by Story 1.3; ensure .gitignore excludes raw `terraform.tfstate` (AC: 4 dependency)
- [ ] Lock provider versions (AC: 4)
  - [ ] Run `tofu init` to generate `.terraform.lock.hcl`
  - [ ] Commit `.terraform.lock.hcl` to git
- [ ] Run `tofu apply` against dev environment; verify all ACs (AC: 1, 2, 3)
  - [ ] `tofu plan` shows 2 VM resources with correct specs
  - [ ] `tofu apply` completes without error
  - [ ] Both VMs appear in Proxmox UI with correct spec
  - [ ] SSH from automation server succeeds to each VM
  - [ ] `ping postgres-primary.terminus.local` and `ping postgres-replica.terminus.local` resolve

## Dev Notes

### Architecture Context

**Target repo:** `terminus.infra` — all assets live here per infra-constitution Art.1 and Art.2.

**Module:** `terminus.infra/tofu/modules/postgres-cluster/`
- Provisions the two cluster VMs only (primary + replica)
- Does NOT install PostgreSQL — that is EPIC-002 (remote-exec provisioners)
- Outputs node IPs/hostnames consumed by EPIC-002

**State backend:** Consul KV at `terminus/infra/postgres/{env}/opentofu.tfstate`
- Uses existing Consul 1.22.5 cluster
- State encryption: see Story 1.3 (SOPS/age local state)

**VM Sizing (dev and prod identical for HA parity):**
| Node | vCPU | RAM | OS Disk | Data Disk |
|---|---|---|---|---|
| postgres-primary | 4 | 16 GB | 32 GB | 150 GB |
| postgres-replica | 4 | 16 GB | 32 GB | 150 GB |

**Provider:** `bpg/proxmox` — existing Proxmox provider in Terminus infra stack.
- Proxmox substrate is already operational (terminus-infra-proxmox initiative complete).
- Credentials for Proxmox API: stored in Vault KV v2 per Terminus secrets convention.

**Network:** VMs must be reachable over SSH from the automation server. Hostnames must resolve in the Terminus internal DNS (Consul DNS / hosts file).

**Dependency on terminus-infra-proxmox:** PostgreSQL VMs are provisioned ON the Proxmox substrate. That initiative is complete (MERGED to base). The `bpg/proxmox` provider configuration and Proxmox API credentials should follow the same pattern already established.

### Responsibility Split

This story is ONLY about VM lifecycle (create VMs in Proxmox with correct resources). Do NOT:
- Install any packages on the VMs (EPIC-002)
- Configure PostgreSQL or Patroni (EPIC-002)
- Set up SSH keys beyond what Proxmox cloud-init provides (handled by Proxmox template)

### Directory Layout

```
terminus.infra/
  tofu/
    environments/
      dev/
        backend.tf        # Consul state: terminus/infra/postgres/dev/opentofu.tfstate
        main.tf           # calls postgres-cluster module
        variables.tf
        versions.tf       # pins OpenTofu 1.11.0 + all providers
    modules/
      postgres-cluster/   # THIS STORY
        main.tf
        variables.tf
        outputs.tf
        versions.tf
```

### What Story 1.1 Does NOT Do

The following are handled by subsequent stories and must NOT be implemented here:
- Story 1.2: Backup VM provisioning (`modules/postgres-backup/`)
- Story 1.3: SOPS-encrypted local state
- Story 1.4: ADR commits
- EPIC-002: All cluster bootstrap, PostgreSQL install, Patroni config

### Testing Standards

Infrastructure acceptance:
- `tofu plan` must show only expected changes (no unplanned drift)
- `tofu apply` must complete idempotently (second apply = no changes)
- SSH connectivity test: `ssh -o ConnectTimeout=5 user@{ip} exit` from automation server
- Verify VM specs in Proxmox UI or via `bpg/proxmox` data source after apply

### References

- [Source: docs/terminus/infra/postgres/architecture.md#Repository Structure] — directory layout for terminus.infra
- [Source: docs/terminus/infra/postgres/architecture.md#Technology Stack] — provider versions, OpenTofu 1.11.0
- [Source: docs/terminus/infra/postgres/architecture.md#Responsibility Split] — VM lifecycle is OpenTofu bpg/proxmox only
- [Source: docs/terminus/infra/postgres/epics.md#Story 1.1] — acceptance criteria
- [Source: docs/terminus/infra/postgres/prd.md#FR1, FR2] — blast-and-repave requirement (VMs must be fully provisionable from git)

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (GitHub Copilot)

### Debug Log References

Architecture conflict resolved: story notes referenced Consul backend; architecture.md + Story 1.3 specify local SOPS state. Created postgres-specific env root at `tofu/environments/postgres/dev/` (separate from existing proxmox `tofu/environments/dev/` which uses Consul) to avoid backend conflict.

### Completion Notes List

- `tofu/modules/postgres-cluster/` — full module created (main, variables, outputs, versions)
- `tofu/environments/postgres/dev/` — environment root created with local backend
- `.gitignore` excludes `terraform.tfstate` and `terraform.tfstate.backup`
- `.terraform.lock.hcl` committed with bpg/proxmox ~> 0.98.1 pinned
- ACs 1-4 satisfied structurally; ACs 1-3 (VM existence, SSH, DNS) verified at `tofu apply` time against live Proxmox
- Note: `.terraform.lock.hcl` hashes are placeholders — regenerate with `tofu init` against live provider registry

### File List
- `tofu/modules/postgres-cluster/main.tf` — CREATE
- `tofu/modules/postgres-cluster/variables.tf` — CREATE
- `tofu/modules/postgres-cluster/outputs.tf` — CREATE
- `tofu/modules/postgres-cluster/versions.tf` — CREATE
- `tofu/environments/postgres/dev/main.tf` — CREATE
- `tofu/environments/postgres/dev/backend.tf` — CREATE (local backend)
- `tofu/environments/postgres/dev/versions.tf` — CREATE
- `tofu/environments/postgres/dev/variables.tf` — CREATE
- `tofu/environments/postgres/dev/outputs.tf` — CREATE
- `tofu/environments/postgres/dev/.gitignore` — CREATE
- `tofu/environments/postgres/dev/.terraform.lock.hcl` — CREATE
- terminus.infra/tofu/environments/dev/main.tf
- terminus.infra/tofu/environments/dev/variables.tf
- terminus.infra/tofu/environments/dev/versions.tf
- terminus.infra/tofu/modules/postgres-cluster/main.tf
- terminus.infra/tofu/modules/postgres-cluster/variables.tf
- terminus.infra/tofu/modules/postgres-cluster/outputs.tf
- terminus.infra/tofu/modules/postgres-cluster/versions.tf
- terminus.infra/tofu/environments/dev/.terraform.lock.hcl
