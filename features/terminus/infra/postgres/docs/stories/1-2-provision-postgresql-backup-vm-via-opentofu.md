# Story 1.2: Provision PostgreSQL backup VM via OpenTofu

Status: ready-for-dev

## Story

As an operator, I want to provision the backup VM by running `tofu apply`, so that I have an isolated, correctly-sized storage host for pgBackRest before backup configuration begins.

## Acceptance Criteria

1. `postgres-backup-vm` exists in Proxmox: 2 vCPU / 4 GB RAM / 300 GB storage
2. VM is reachable over SSH from both cluster nodes
3. 300 GB volume is mounted at `/var/lib/pgbackrest`
4. Module outputs backup VM IP (consumed by `remote-exec` provisioners in EPIC-003)

## Tasks / Subtasks

- [ ] Create `tofu/modules/postgres-backup/` module directory (AC: 1, 4)
  - [ ] Write `main.tf` — `bpg/proxmox` VM resource with 2 vCPU / 4 GB / 300 GB disk
  - [ ] Write `variables.tf` — proxmox_node, vm_name, ssh_key, network config inputs
  - [ ] Write `outputs.tf` — export `backup_vm_ip` output consumed downstream
- [ ] Configure data disk mount in `remote-exec` provisioner (AC: 3)
  - [ ] Format and mount 300 GB disk at `/var/lib/pgbackrest`
  - [ ] Add `/etc/fstab` entry for persistence across reboots
- [ ] Wire backup module into `tofu/environments/dev/main.tf` (AC: 1)
  - [ ] Add `module "postgres_backup"` block referencing `modules/postgres-backup`
- [ ] Verify SSH reachability from cluster node IPs (AC: 2)
  - [ ] SSH test via `remote-exec` null_resource on cluster nodes targeting backup VM IP
- [ ] Run `tofu apply` in dev environment and confirm VM provisioned (AC: 1, 2, 3, 4)

## Dev Notes

### Architecture Context

This story provisions the third VM in the postgres topology. The `postgres-backup-vm` is isolated from the two cluster VMs (NFR9) and hosts the pgBackRest repository used in EPIC-003.

- Module path: `tofu/modules/postgres-backup/`
- Environment: `tofu/environments/dev/`
- Provider: `bpg/proxmox` (same as postgres-cluster module from Story 1.1)
- VM spec: 2 vCPU / 4 GB RAM / 32 GB OS + 300 GB pgBackRest data disk

### Responsibility Split

**This story does:**
- Provisions `postgres-backup-vm` via OpenTofu
- Mounts 300 GB volume at `/var/lib/pgbackrest`
- Exports `backup_vm_ip` module output

**This story does NOT:**
- Install pgBackRest (EPIC-003, Story 3.1)
- Initialize pgBackRest stanza (Story 3.2)
- Encrypt or commit state (Story 1.3 handles SOPS state for all modules)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-backup/
        main.tf          # Proxmox VM resource
        variables.tf     # Input variables
        outputs.tf       # backup_vm_ip output
    environments/
      dev/
        main.tf          # Adds module "postgres_backup" block
```

### Testing Standards

- `tofu plan` shows only the backup VM resource being created (no drift on cluster VMs)
- `tofu apply` completes without error
- `ssh postgres-backup-vm` from automation server succeeds
- `df -h /var/lib/pgbackrest` shows 300 GB volume mounted
- `tofu output backup_vm_ip` returns a valid IP address

### References

- epics.md: Story 1.2 Acceptance Criteria
- architecture.md: VM sizing section — postgres-backup-vm: 2 vCPU / 4 GB / 300 GB
- architecture.md: NFR9 — Backup storage isolation from cluster nodes

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-backup/main.tf` — CREATE
- `tofu/modules/postgres-backup/variables.tf` — CREATE
- `tofu/modules/postgres-backup/outputs.tf` — CREATE
- `tofu/environments/dev/main.tf` — MODIFY (add postgres_backup module block)
