# Technical Decisions Log

**Initiative:** terminus-infra-proxmox
**Phase:** TechPlan
**Date:** 2026-03-21
**Status:** Approved for implementation planning

## Summary

This document captures the concrete technical decisions that govern implementation of the Proxmox infrastructure initiative. It is the execution-focused companion to the architecture document and should be treated as the default decision record for repository bootstrap, module design, secret handling, and operational workflow setup.

## Decision Register

### TD-001: Use OpenTofu as the Provisioning Authority

- Status: accepted
- Decision: Use OpenTofu as the system of record for declarative infrastructure lifecycle.
- Version: 1.11.0
- Rationale: The initiative is infrastructure-focused and needs explicit state, reproducible plans, and drift-resistant provisioning. OpenTofu fits the repo's greenfield state and avoids mixing provisioning concerns into Ansible.
- Consequences:
  - Infrastructure topology lives under `tofu/`
  - Plans and applies are environment-scoped
  - Ansible does not define authoritative infrastructure topology

### TD-002: Use `bpg/proxmox` for Proxmox Control

- Status: accepted
- Decision: Use the `bpg/proxmox` provider for Proxmox resource management.
- Version: 0.98.1
- Rationale: It is the chosen provider in the architecture and aligns with the goal of keeping Proxmox lifecycle in OpenTofu.
- Consequences:
  - Proxmox cluster, network, and storage resources are modeled in OpenTofu modules
  - Provider compatibility becomes a tracked dependency for implementation stories

### TD-003: Store Remote State in Consul

- Status: accepted
- Decision: Use Consul as the OpenTofu remote state backend.
- Version: 1.22.5
- Rationale: Team-safe shared state and locking are required. Consul supports environment-scoped backend separation and avoids local state drift.
- Consequences:
  - Each environment must have an explicit backend configuration
  - State key naming must remain environment-scoped and predictable
  - Bootstrap must establish remote state before infrastructure apply

### TD-004: Keep Configuration Management in Ansible

- Status: accepted
- Decision: Use Ansible for guest bootstrap and operational automation only.
- Version: ansible-core 2.19 / Ansible 12
- Related collection: `community.proxmox` 1.5.0
- Rationale: Ansible is strong for configuration and operations, but it should not compete with OpenTofu on infrastructure ownership.
- Consequences:
  - Ansible consumes OpenTofu outputs or rendered inventory
  - Bootstrap and day-2 operations live under `ansible/`
  - Ansible should not scrape raw state files directly

### TD-005: Use SOPS for Repo-Managed Secret Source Files

- Status: accepted
- Decision: Store repo-managed secret source material as SOPS-encrypted YAML files.
- Rationale: The workflow needs Git-native encrypted source control without plaintext secrets in the repository.
- Consequences:
  - Encrypted source files live under `secrets/<environment>/<capability>/`
  - Secret filenames must remain semantic and stable
  - Secret changes remain reviewable in Git without exposing plaintext

### TD-006: Use `age` for SOPS Key Management

- Status: accepted
- Decision: Use `age` as the SOPS key model.
- Rationale: The architecture selected `age` for a simpler and cleaner bootstrap path than more complex alternatives.
- Consequences:
  - Initial bootstrap must define key custody and distribution rules
  - `.sops.yaml` becomes a required repository root file

### TD-007: Use Vault KV v2 for Runtime Secret Delivery

- Status: accepted
- Decision: Use Vault KV v2 as the runtime secret authority.
- Version: 1.21.4
- Rationale: Runtime consumers should not depend directly on repo decryption. Vault KV v2 provides versioned operational secret delivery.
- Consequences:
  - Secret publication becomes an explicit environment-scoped workflow
  - Vault paths follow an environment-first taxonomy such as `secret/terminus/dev/proxmox/...`
  - Vault Transit is not required in the initial implementation

### TD-008: Use Per-Environment Machine Identities

- Status: accepted
- Decision: Separate automation identities by environment.
- Rationale: Environment trust boundaries are first-class and should not be collapsed into a single shared platform identity.
- Consequences:
  - Vault policies and secret paths must align with environment boundaries
  - Environment roots remain isolated for state, secrets, and automation execution

### TD-009: Use a Hybrid Repository Composition Model

- Status: accepted
- Decision: Organize the repository as shared modules plus environment-root composition.
- Rationale: The repo needs reusable primitives without duplicating full definitions per environment.
- Consequences:
  - Shared capability logic lives in `tofu/modules/`
  - Environment-specific assembly lives only under `tofu/environments/`
  - Duplication across environment roots is treated as a smell unless justified

### TD-010: OpenTofu Outputs Are the Integration Handoff Contract

- Status: accepted
- Decision: Use explicit OpenTofu outputs as the primary handoff mechanism to Ansible and auxiliary automation.
- Rationale: Direct state parsing is brittle and bypasses architectural boundaries.
- Consequences:
  - Output names use descriptive `snake_case`
  - Inventory generation and bootstrap inputs should be deterministic and machine-readable

### TD-011: No API Contract Artifact for Day-One Scope

- Status: accepted
- Decision: Do not create `api-contracts.md` for the current initiative scope.
- Rationale: The architecture is control-plane oriented and does not define a user-facing or service-facing application API at this phase.
- Consequences:
  - The techplan artifact set for this initiative is `architecture.md` plus `tech-decisions.md`
  - If a later capability introduces an operator or service API, create a dedicated API contracts artifact then

## Deferred Decisions

These decisions were intentionally left for later planning or implementation stories:

- Exact Proxmox topology and node layout
- Exact datastore and storage policy strategy
- Exact network segmentation and SDN layout
- Exact Consul state key naming convention
- Exact Vault path taxonomy and policy granularity
- CI/CD workflow realization details
- Observability, backup, and disaster recovery implementation specifics

## Implementation Constraints

- Keep provisioning, configuration, state coordination, and runtime secret delivery as separate concerns
- Use lowercase kebab-case for directories and `snake_case` for OpenTofu variables and outputs
- Keep environment assembly only in environment roots
- Do not introduce plaintext secret files into the repository
- Do not let Ansible become a second infrastructure authority

## Initial Execution Order

1. Create the repository skeleton under `tofu/`, `ansible/`, `secrets/`, `docs/`, `scripts/`, and `tests/`
2. Establish Consul-backed environment roots
3. Introduce the `bpg/proxmox` provider and shared modules
4. Add `.sops.yaml` and initial encrypted secret files
5. Define Vault KV v2 publication workflow and per-environment access model
6. Add Ansible bootstrap and operational playbooks that consume explicit OpenTofu outputs

## Source of Truth

- Architecture baseline: `architecture.md`
- This document: implementation-level technical decision record
- Any future exceptions should be captured as ADRs under the target repository `docs/decisions/`