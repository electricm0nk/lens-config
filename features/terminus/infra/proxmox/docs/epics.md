stepsCompleted:
  - 1
  - 2
  - 3
  - 4
inputDocuments:
  - docs/terminus/infra/proxmox/technical-requirements.md
  - docs/terminus/infra/proxmox/architecture.md
  - docs/terminus/infra/proxmox/tech-decisions.md
  - docs/terminus/infra/proxmox/prd.md
project_name: proxmox
user_name: CrisWeber
date: '2026-03-22'
workflowType: 'epics-and-stories'
status: 'complete'
lastStep: 4
---

# proxmox — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the `terminus-infra-proxmox` initiative, decomposing requirements from the technical requirements, architecture, and tech-decisions log into implementable stories.

**Track:** tech-change (no preplan/businessplan artifacts — architecture-first)  
**Audience:** medium (devproposal phase)  
**Initiative Root:** terminus-infra-proxmox

---

## Requirements Inventory

### Functional Requirements

FR1: Manage Proxmox resource lifecycle (cluster, network, storage) declaratively via OpenTofu using the `bpg/proxmox` provider (v0.98.1)

FR2: Configure and manage OpenTofu remote state per environment using a Consul (v1.22.5) backend with environment-scoped state keys

FR3: Implement SOPS-encrypted secret authoring in Git using `age` for key management — all repo-held secret source material must be encrypted before commit

FR4: Deliver runtime secrets to deployed systems via Vault KV v2 (Vault v1.21.4) using an explicit environment-first Vault path taxonomy

FR5: Implement Ansible (ansible-core 2.19) guest bootstrap and post-provision operational automation that consumes OpenTofu outputs and Vault-delivered secrets (not raw state files)

FR6: Create and document per-environment machine identities for Proxmox API and infrastructure automation, with distinct dev and prod trust boundaries

FR7: Define and document the Consul remote state key naming convention so all environment roots follow a predictable, environment-scoped naming pattern

FR8: Define and document the Vault KV v2 path taxonomy so all secret publication targets follow a consistent, environment-first hierarchy

FR9: Implement OpenTofu-to-Ansible handoff via deterministic outputs or rendered inventory artifacts — Ansible must not parse raw tfstate directly

FR10: Provide end-to-end bootstrap validation: Consul state configured → infrastructure provisioned → secrets published to Vault → Ansible bootstrap completes

---

### Non-Functional Requirements

NFR1: **Security** — No secrets may be stored in plaintext in source control under any circumstances (org constitution Article 9)

NFR2: **TDD Red-Green Discipline** — All production code (scripts, playbooks, OpenTofu modules) must follow red-green TDD: failing test precedes implementation (org constitution Article 7)

NFR3: **BDD Acceptance Criteria** — Every story acceptance criterion must have a fully implemented, non-stub BDD test. No pending, skip, or empty test bodies permitted (org constitution Article 8)

NFR4: **Infrastructure Ownership** — OpenTofu is the sole authority for Proxmox infrastructure topology. Ansible must not define or duplicate source-of-truth infrastructure state

NFR5: **Environment Isolation** — State, identity, inventory, and Vault path scopes must be environment-scoped. No shared global state across dev and prod

NFR6: **Idempotency** — Vault publication playbooks and OpenTofu apply runs must be idempotent and produce consistent results across repeated executions

NFR7: **Drift Control** — Remote Consul backend is the state authority. Local state file parsing is prohibited in all automation scripts and playbooks

NFR8: **Naming Consistency** — Directories: lowercase kebab-case. OpenTofu variables, outputs, and resource names: `snake_case`. Environment names: lowercase (`dev`, `prod`). Vault paths: slash-delimited lowercase with environment before capability

NFR9: **Repo Boundary** — All Proxmox infra features belong in the shared `terminus.infra` repository unless an explicit repository-boundary exception is documented and approved (terminus constitution Article 1; infra constitution Article 1)

---

### Additional Requirements

- **Bootstrap sequence ordering (REQUIRED):** Remote state (Consul) must be configured before infrastructure apply; infrastructure must be provisioned before secrets are published; secrets must be published before Ansible bootstrap runs. Implementation stories must enforce this ordering.

- **Repository skeleton bootstrap:** Repo must be initialized with `tofu/modules/`, `tofu/environments/{dev,prod}/`, `ansible/{inventories,playbooks,roles}/`, `secrets/{dev,prod}/proxmox/`, `docs/`, `scripts/`, `tests/` before any capability work begins.

- **SOPS/age key custody:** An explicit operator action is required to generate and distribute the `age` key(s). The `.sops.yaml` root file must be present before any SOPS-encrypted file is committed. Key custody plan must be documented before the secret workflow story begins.

- **Deferred decisions to resolve early:** (1) Consul state key naming convention, (2) Vault path taxonomy. These must be locked in a sprint-zero prerequisite story before dependent implementation stories begin.

- **Tech-decisions compliance:** TD-001 through TD-008 are accepted decisions. Implementation stories must not contradict or deviate from these decisions without a new decision record.

---

### FR Coverage Map

FR1: Epic 2 - OpenTofu-managed Proxmox provisioning authority and shared module layout
FR2: Epic 2 - Environment-scoped Consul backend configuration for dev and prod
FR3: Epic 1, Epic 3 - `.sops.yaml`, `age` key bootstrap, and encrypted secret source files
FR4: Epic 1, Epic 3 - Vault KV v2 path taxonomy and runtime secret publication workflow
FR5: Epic 4 - Ansible bootstrap and post-provision operational automation
FR6: Epic 2 - Per-environment automation identities and trust-boundary setup
FR7: Epic 1 - Consul state key naming convention and bootstrap decision record
FR8: Epic 1 - Vault path taxonomy decision record and repo-wide convention
FR9: Epic 2, Epic 4 - Deterministic OpenTofu outputs and inventory handoff into Ansible
FR10: Epic 5 - End-to-end validation of the full bootstrap chain

---

## Epic List

### Epic 1: Repository Foundation & Decision Prerequisites
The operator can initialize the `terminus.infra` repository skeleton, lock the Consul state key naming convention, lock the Vault KV v2 path taxonomy, and establish the `.sops.yaml` plus `age` key bootstrap model so all later implementation work starts from a consistent control-plane baseline.
**FRs covered:** FR3, FR4, FR7, FR8

### Epic 2: Infrastructure Provisioning With OpenTofu
The operator can provision Proxmox infrastructure declaratively from environment roots using OpenTofu with Consul-backed remote state, per-environment identities, and deterministic outputs that downstream automation can consume safely.
**FRs covered:** FR1, FR2, FR6, FR9

### Epic 3: Secret Authoring & Vault Publication
The operator can author encrypted secret source files with SOPS + `age`, publish the required runtime values into Vault KV v2 using the approved taxonomy, and repeat that publication idempotently for each environment.
**FRs covered:** FR3, FR4

### Epic 4: Guest Bootstrap & Operational Automation
The operator can render inventory from OpenTofu outputs and run Ansible bootstrap and day-2 operational playbooks using Vault-delivered secrets without parsing raw state files.
**FRs covered:** FR5, FR9

### Epic 5: End-to-End Validation & Operator Readiness
The operator can execute and verify the full bootstrap chain in a dev environment and use a complete runbook to recover, re-run, and diagnose failures without reading implementation code.
**FRs covered:** FR10

---

## Epic 1: Repository Foundation & Decision Prerequisites

The operator can initialize the repository skeleton and freeze the control-plane conventions that every later story depends on.

### Story 1.1: Bootstrap Repository Skeleton And Quality Baseline

**FRs implemented:** FR7

As an operator,
I want the `terminus.infra` repository initialized with the required directory structure and validation entry points,
So that all later infrastructure work starts from a verified, convention-compliant baseline.

**Acceptance Criteria:**

1. The repository contains the architecture-defined top-level directories: `docs/`, `tofu/`, `ansible/`, `secrets/`, `scripts/`, `tests/`, and `.github/workflows/`
2. Placeholder README files exist for the root, `tofu/modules/`, `tofu/environments/dev/`, `tofu/environments/prod/`, `ansible/playbooks/`, and `secrets/`
3. Validation entry points exist for `tests/tofu/`, `tests/ansible/`, and `scripts/validate.sh`
4. A failing-first test or validation check is committed before any baseline implementation files required by this story
5. Naming and structure in the repo baseline follow the architecture conventions exactly

### Story 1.2: Lock Consul State And Vault Path Conventions

**FRs implemented:** FR7, FR8

As an operator,
I want the Consul state key naming convention and Vault KV v2 path taxonomy documented in a committed decision record,
So that every environment and automation path uses a single predictable convention.

**Acceptance Criteria:**

1. A decision document defines the Consul backend key pattern for all environments
2. A decision document defines the environment-first Vault KV v2 path taxonomy for all Proxmox secret material
3. The documented conventions match the architecture constraints and do not conflict with existing terminus governance
4. Story 2 and Story 3 implementation inputs can reference these conventions without ambiguity
5. The decision record includes at least one concrete example for both `dev` and `prod`

### Story 1.3: Establish SOPS Bootstrap And Age Key Custody

**FRs implemented:** FR3, FR4

As an operator,
I want `.sops.yaml`, the `age` recipient model, and operator key-custody steps defined,
So that encrypted secrets can be created safely without ad hoc manual handling.

**Acceptance Criteria:**

1. `.sops.yaml` exists at the repository root and maps the approved secret file locations
2. The operator bootstrap document defines how `age` keys are created, stored, and shared with authorized automation and humans
3. An example encrypted secret file path is defined for both `dev` and `prod`
4. The custody procedure makes clear that plaintext secrets are never committed to Git
5. A validation check fails when an expected secret file is unencrypted or placed outside the approved path pattern

---

## Epic 2: Infrastructure Provisioning With OpenTofu

The operator can provision environment-scoped Proxmox infrastructure using OpenTofu as the sole topology authority.

### Story 2.1: Define Environment Roots And Consul Backends

**FRs implemented:** FR2

As an operator,
I want `dev` and `prod` OpenTofu environment roots with remote Consul backend configuration,
So that infrastructure state is shared safely and isolated per environment.

**Acceptance Criteria:**

1. `tofu/environments/dev/` and `tofu/environments/prod/` each contain backend, provider, version, main, variable, and output files
2. Each environment root uses the approved Consul key naming convention from Story 1.2
3. Local state files are excluded from source control and not required for normal operation
4. Validation commands succeed for both environment roots with no backend naming ambiguity
5. Tests or validation scripts for backend configuration are written before the final implementation for this story

### Story 2.2: Implement Proxmox Module Skeleton And Environment Composition

**FRs implemented:** FR1

As an operator,
I want reusable OpenTofu modules and environment composition for Proxmox cluster, network, and storage concerns,
So that infrastructure can be planned and applied declaratively without duplicating topology definitions per environment.

**Acceptance Criteria:**

1. Shared modules exist for the agreed Proxmox capabilities needed by the baseline architecture
2. Environment roots compose shared modules rather than duplicating complete infrastructure definitions inline
3. The `bpg/proxmox` provider is configured in a way compatible with the approved architecture decisions
4. Module inputs and outputs use `snake_case` naming consistently
5. Validation coverage exists for module structure and basic variable contracts before implementation is finalized

### Story 2.3: Add Environment Identities And Deterministic Outputs

**FRs implemented:** FR6, FR9

As an operator,
I want per-environment automation identities and deterministic OpenTofu outputs,
So that downstream Ansible workflows can consume infrastructure metadata without reading raw state.

**Acceptance Criteria:**

1. Dev and prod identity inputs are defined separately and documented as distinct trust boundaries
2. Output values needed by downstream automation are exposed from the environment roots with stable names
3. No script or playbook in the story parses raw tfstate to obtain provisioning data
4. Output contracts cover at least host addressing, inventory grouping inputs, and secret publication context required downstream
5. Tests or contract checks exist for output names and required fields before final implementation is completed

---

## Epic 3: Secret Authoring & Vault Publication

The operator can manage encrypted secret source material in Git and publish runtime values to Vault using the approved taxonomy.

### Story 3.1: Create Encrypted Secret Source Layout

**FRs implemented:** FR3

As an operator,
I want environment-scoped SOPS-encrypted secret source files for Proxmox automation,
So that sensitive values are versioned safely and remain separated by environment.

**Acceptance Criteria:**

1. `secrets/dev/proxmox/` and `secrets/prod/proxmox/` contain the approved encrypted file skeletons
2. Secret filenames are semantic and match the taxonomy defined in Story 1.2 and Story 1.3
3. No plaintext secrets are present in tracked files created or modified by this story
4. Validation catches malformed or missing encrypted secret files in the approved locations
5. The secret source layout is sufficient to support Vault publication in the next story without renaming

### Story 3.2: Implement Vault Publication Workflow

**FRs implemented:** FR4

As an operator,
I want a repeatable publish workflow that maps encrypted source files into Vault KV v2 paths,
So that runtime consumers receive the required secret material from Vault rather than from Git decryption at execution time.

**Acceptance Criteria:**

1. A dedicated publication playbook or script reads approved encrypted inputs and writes to Vault KV v2 using the documented taxonomy
2. Source-to-destination mapping is explicit and environment-scoped
3. Publication failures stop the workflow and surface actionable errors
4. The workflow does not write secrets to logs, temp files, or source-controlled artifacts
5. Tests or validation checks exist for path mapping and failure handling before final implementation is completed

### Story 3.3: Verify Idempotent Secret Publication In Dev

**FRs implemented:** FR4

As an operator,
I want the Vault publication workflow to be safe to run repeatedly in `dev`,
So that routine updates do not introduce drift or duplicate manual repair steps.

**Acceptance Criteria:**

1. Re-running the publish workflow in `dev` updates secrets deterministically without destructive side effects
2. The workflow reports which environment and capability it is targeting in structured, operator-readable output
3. Validation proves that repeated runs do not require changes when inputs are unchanged
4. Error handling distinguishes missing source secrets from Vault connectivity or authorization failures
5. The story records the expected operator command path for future reuse in the runbook

---

## Epic 4: Guest Bootstrap & Operational Automation

The operator can bootstrap and operate provisioned Proxmox resources using Ansible with clear handoff contracts from OpenTofu.

### Story 4.1: Render Inventory From OpenTofu Outputs

**FRs implemented:** FR9

As an operator,
I want generated inventory or structured bootstrap inputs derived from OpenTofu outputs,
So that Ansible can target the provisioned infrastructure without scraping raw state.

**Acceptance Criteria:**

1. Inventory generation consumes only documented OpenTofu outputs from Story 2.3
2. The generated inventory path and naming are deterministic and environment-scoped
3. Inventory structure is valid for the baseline bootstrap playbooks
4. Inventory generation fails clearly if required outputs are missing
5. Tests or validation checks for inventory rendering are written before the final implementation for this story

### Story 4.2: Implement Baseline Guest Bootstrap Playbook

**FRs implemented:** FR5

As an operator,
I want an Ansible bootstrap playbook and reusable roles for baseline guest setup,
So that newly provisioned infrastructure can be brought into an operational baseline with approved secret delivery.

**Acceptance Criteria:**

1. A bootstrap playbook exists and uses the generated inventory from Story 4.1
2. The playbook consumes Vault-delivered secrets rather than plaintext repo values
3. Bootstrap responsibilities remain within Ansible and do not redefine infrastructure topology already owned by OpenTofu
4. Playbook syntax and inventory validation pass before the implementation is considered complete
5. The bootstrap role structure aligns with the architecture-defined boundaries under `ansible/roles/`

### Story 4.3: Add Day-2 Operational Playbooks And Validation

**FRs implemented:** FR5

As an operator,
I want day-2 operational playbooks for validation and repeatable maintenance actions,
So that the infra service has a supported operational workflow after initial bootstrap.

**Acceptance Criteria:**

1. At least one validation-oriented playbook exists for environment verification after bootstrap
2. Operational playbooks consume the same inventory and secret patterns as bootstrap
3. Operational actions are scoped to infra-service concerns and do not duplicate OpenTofu provisioning logic
4. Playbook failure modes are documented with actionable operator guidance
5. Validation and syntax checks are present for each added operational playbook

---

## Epic 5: End-to-End Validation & Operator Readiness

The operator can validate the full bootstrap chain and recover it from the documented runbook.

### Story 5.1: Build End-To-End Validation Harness

**FRs implemented:** FR10

As an operator,
I want a single validation entry point that checks the full bootstrap chain,
So that I can prove the environment is ready without manually stitching together each tool invocation.

**Acceptance Criteria:**

1. A single validation entry point verifies backend readiness, provisioning prerequisites, secret publication prerequisites, inventory generation, and bootstrap readiness
2. Validation output identifies the failing stage clearly when the chain is broken
3. The harness does not mutate production infrastructure as part of validation-only execution
4. Tests for the validation harness are written before final implementation changes for this story
5. The validation entry point is documented as the standard readiness check for future operators

### Story 5.2: Document Operator Runbook And Recovery Paths

**FRs implemented:** FR10

As a returning operator,
I want a complete runbook for setup, execution, recovery, and repeated operation,
So that I can restore service or continue planned work without needing source-level tribal knowledge.

**Acceptance Criteria:**

1. The runbook documents initial setup, validation, provisioning flow, secret publication flow, bootstrap flow, and common failure recovery
2. The runbook explains the required operator-managed steps for `age` key custody and Vault access
3. The runbook includes the standard commands for `dev` environment bootstrap and validation
4. The runbook identifies where logs, outputs, and inventory artifacts should be inspected during troubleshooting
5. A reader can follow the runbook without consulting implementation files to understand the expected operating sequence

### Story 5.3: Execute Dev Smoke Validation And Capture Readiness Evidence

**FRs implemented:** FR10

As an operator,
I want a documented dev smoke validation run,
So that the full chain is proven once before later sprint planning or execution work depends on it.

**Acceptance Criteria:**

1. The `dev` environment smoke flow executes the approved validation sequence end to end
2. Evidence of success or failure is captured in a repeatable location referenced by the runbook
3. Any unresolved failure is classified by stage: backend, provisioning, secret publication, inventory, or bootstrap
4. The smoke validation does not require undocumented manual steps
5. The resulting evidence is sufficient to support implementation-readiness review for the initiative

---
