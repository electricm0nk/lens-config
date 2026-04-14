---
stepsCompleted:
  - 1
  - 2
  - 3
  - 4
  - 5
  - 6
  - 7
inputDocuments:
  - _bmad-output/lens-work/initiatives/terminus/infra/phases/techplan/prd.md
  - _bmad-output/lens-work/initiatives/terminus/infra/phases/techplan/technical-requirements.md
workflowType: 'architecture'
project_name: 'proxmox'
user_name: 'CrisWeber'
date: '2026-03-21'
lastStep: 8
status: 'complete'
completedAt: '2026-03-21'

# Architecture Decision Document

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements:**
The current requirements define an infrastructure-focused Proxmox initiative rather than an end-user feature. Architecturally, the work must:
- Define the Proxmox-related technical change at the infrastructure layer
- Clarify what capabilities, resources, or standards are being introduced
- Identify the systems, services, or automation that will integrate with Proxmox
- Establish implementation boundaries and dependency edges clearly enough to support downstream planning

The current inputs also imply several architectural questions that must be resolved early because they shape the solution structure:
- What operational problem is this initiative primarily solving
- What Proxmox resources or capabilities are in scope
- What the intended automation boundary is
- What the unit of implementation change will be, such as cluster configuration, provisioning workflows, templates, networking, storage, backup, or policy controls

There are no epics or user stories yet, so the architecture will need to provide the structural decomposition that later planning can turn into execution work.

**Non-Functional Requirements:**
The current inputs emphasize non-functional quality over feature behavior. The architecture must make the following explicit:
- Security considerations
- Operational expectations
- Failure modes and recovery expectations
- Storage constraints
- Network constraints
- Identity and secret-management constraints
- Sufficient clarity to support implementation without a business-planning phase

These requirements suggest the design will be judged primarily on operability, safety, dependency clarity, implementation readiness, and resistance to configuration drift.

**Scale & Complexity:**
This initiative appears to be a medium-complexity infrastructure effort. It does not show product or UX complexity, but it does show meaningful platform complexity because the design must coordinate several cross-cutting infrastructure domains while still leaving room for later execution planning.

- Primary domain: infrastructure/platform architecture
- Complexity level: medium
- Estimated architectural components: cluster/control plane, automation/integration, network, identity/secrets, storage, observability, operations/recovery

### Technical Constraints & Dependencies

Known constraints and dependencies from the current documents include:
- The initiative is Proxmox-focused and infrastructure-only
- No business-plan or UX artifacts are required for this track
- The resulting architecture must be detailed enough to support later execution planning directly
- The solution must account for integrations with external systems, services, or automation, though those integrations are not yet enumerated
- Network, identity, secret-management, and storage concerns are first-class design constraints

Important unknowns that should be treated as architectural inputs:
- Whether the initiative is centered on provisioning, standardization, operations, tenancy, resilience, or lifecycle management
- What automation systems will be authoritative
- Where configuration state will live and how drift will be managed
- What administrative and operational trust boundaries exist
- What failure domains must be isolated

### Cross-Cutting Concerns Identified

The concerns likely to affect multiple architectural components are:
- Security and access control
- Administrative trust boundaries
- Network topology and segmentation
- Secret handling and credential flows
- Storage architecture and lifecycle expectations
- Automation ownership and integration patterns
- State management and drift control
- Operational observability and supportability
- Failure handling, resilience, backup, and recovery procedures

## Starter Template Evaluation

### Primary Technology Domain

Infrastructure/platform architecture based on project requirements analysis and the greenfield state of the `terminus.infra` repository.

### Starter Options Considered

Because this initiative is an infrastructure service rather than an application, no conventional application starter template is a strong fit. Instead, the relevant foundation options are infrastructure toolchain starters:

- **OpenTofu-first foundation**
  - Best fit for greenfield declarative infrastructure
  - Open governance and broad Terraform-language compatibility
  - Official docs currently show OpenTofu 1.11.0
  - Strong fit if Proxmox infrastructure should be represented as versioned declarative state

- **Terraform-first foundation**
  - Mature and widely adopted
  - Official install docs currently show Terraform 1.14.7
  - Viable if there is an external organizational preference for HashiCorp tooling
  - Slightly weaker default choice here because no existing Terraform estate is documented

- **Ansible-first foundation**
  - Strong for host configuration, operational workflows, and post-provisioning automation
  - Official stable docs currently reference ansible-core 2.19 / Ansible 12
  - Useful, but weaker as the sole foundation if the project needs durable declarative infrastructure state and drift management

- **Pulumi-first foundation**
  - Strong option when the team wants general-purpose languages as the primary IaC interface
  - Official docs confirm active maintenance
  - Less attractive as the default here because the current repo has no language preference or existing Pulumi investment documented

### Selected Starter: Custom Infrastructure Foundation (OpenTofu + Ansible)

**Rationale for Selection:**
This repo is greenfield and infrastructure-focused. The best foundation is not a framework boilerplate but a deliberate split of responsibilities:

- OpenTofu manages declarative infrastructure state and topology
- Ansible handles guest/bootstrap configuration and operational automation
- This keeps infrastructure lifecycle concerns explicit
- This separates provisioning from configuration cleanly
- This aligns well with the project context concerns around drift control, recovery, access boundaries, and automation ownership

This is a better architectural fit than forcing all concerns into a single tool.

**Initialization Command:**

```bash
mkdir -p tofu/environments/{dev,prod} tofu/modules ansible/{inventories,playbooks,roles} docs
```

**Architectural Decisions Provided by Starter:**

**Language & Runtime:**
- Declarative HCL-compatible infrastructure definitions via OpenTofu
- YAML plus Python-based automation runtime via Ansible where configuration management is needed

**Styling Solution:**
- Not applicable to this infrastructure repository

**Build Tooling:**
- OpenTofu CLI workflow for init/plan/apply
- Ansible CLI workflow for inventory, playbooks, and operational runs

**Testing Framework:**
- Validation is expected through `tofu fmt`, `tofu validate`, and Ansible syntax/inventory checks
- Additional policy or integration testing can be layered later as implementation stories

**Code Organization:**
- `tofu/modules` for reusable infrastructure primitives
- `tofu/environments` for environment-specific composition
- `ansible/roles` for reusable configuration logic
- `ansible/playbooks` for task orchestration
- `docs` for architecture, operating rules, and runbooks

**Development Experience:**
- Clear separation between provisioning and configuration
- Easier reasoning about ownership boundaries
- Better support for incremental rollout across planned repo feature areas such as proxmox, k3s, postgres, and secrets

**Note:** Repository initialization and toolchain bootstrap should be the first implementation story.

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Proxmox resource lifecycle will be managed by OpenTofu using the `bpg/proxmox` provider
- OpenTofu state will be stored remotely using a Consul backend
- Repository secret material will be authored and stored using SOPS
- SOPS key management will use `age`
- Runtime secret delivery will use Vault `KV v2`
- Automation identities will be separated per environment
- Repository composition will follow a hybrid model of shared modules plus environment-root assembly

**Important Decisions (Shape Architecture):**
- Ansible is a secondary automation layer used for guest/bootstrap and post-provision operational workflows
- Vault is a runtime secret authority only in the initial architecture; Vault Transit is not a mandatory dependency in the day-one path
- Capability modules will coexist with environment-specific composition roots
- Trust boundaries will align with environment boundaries rather than with a single shared platform identity

**Deferred Decisions (Post-MVP):**
- Detailed Proxmox cluster topology
- Storage class and datastore strategy
- Network segmentation and SDN layout
- Backup and restore workflow specifics
- Observability stack and log/metric sink choice
- CI/CD implementation details
- Exact Vault path taxonomy and policy granularity
- Exact Consul namespace or key layout for state separation

### Data Architecture

This initiative is infrastructure-oriented rather than application-data-oriented, so the primary state model is infrastructure state, configuration state, and secret state.

- Infrastructure state will be represented declaratively in OpenTofu
- Shared state coordination will use a remote Consul backend rather than local state files
- State ownership is environment-scoped, not shared globally across all environments
- Secrets are treated separately from infrastructure state and will not be stored directly in plain OpenTofu configuration
- Runtime secret material will be published into Vault `KV v2`, which provides versioned secret storage suitable for operational use

**Rationale:**
This separates infrastructure lifecycle, secret lifecycle, and runtime access concerns cleanly while supporting team-safe collaboration.

### Authentication & Security

- Machine authentication will use separate per-environment automation identities
- Human and machine trust boundaries should remain distinct
- SOPS with `age` will be the Git-native encryption mechanism for repo-held secrets
- Vault `KV v2` will be the runtime secret source for deployed systems and automation that should not depend directly on repo decryption at execution time
- Vault Transit is intentionally deferred from the required baseline architecture
- Secrets should be authored encrypted in Git, published selectively to Vault, and consumed from Vault at runtime

**Rationale:**
This gives strong isolation between environments while keeping the initial operator workflow relatively simple and auditable.

### API & Communication Patterns

For this initiative, the key communication patterns are between infrastructure control systems rather than application services.

- OpenTofu communicates with Proxmox through the `bpg/proxmox` provider
- Ansible consumes outputs and performs guest/bootstrap or operational tasks after provisioning
- Vault is the runtime secret distribution endpoint
- Consul is the remote state coordination backend for OpenTofu
- No separate application API pattern such as REST or GraphQL is required at this stage

**Rationale:**
The architecture is centered on infrastructure control-plane interactions, not application request/response design.

### Frontend Architecture

Not applicable for this initiative.

There is no user-facing frontend in the current scope. If later operational tooling requires UI surfaces, those should be designed as separate capabilities rather than folded into the base Proxmox infrastructure architecture.

### Infrastructure & Deployment

- Primary provisioning authority: OpenTofu + `bpg/proxmox`
- Secondary automation authority: Ansible for guest/bootstrap and post-provision operations
- Remote state backend: Consul
- Runtime secret store: Vault `KV v2`
- Secret authoring format in Git: SOPS
- Secret key model: `age`
- Repository structure: hybrid model with reusable shared modules and environment-root composition
- Environment trust boundary: per-environment machine identities

**Verified technology references at planning time:**
- OpenTofu: 1.11.0
- Consul: 1.22.5
- Vault: 1.21.4
- Ansible stable line: ansible-core 2.19 / Ansible 12
- `bpg/proxmox` provider: 0.98.1
- `community.proxmox` collection: 1.5.0

**Rationale:**
This stack is intentionally conservative. It prioritizes explicit ownership boundaries, stable infrastructure workflows, and clean separation between provisioning, configuration, state coordination, and secret delivery.

### Decision Impact Analysis

**Implementation Sequence:**
1. Establish repository structure for hybrid composition
2. Define environment roots and shared module boundaries
3. Configure Consul remote state strategy per environment
4. Bootstrap OpenTofu provider usage with `bpg/proxmox`
5. Establish SOPS + `age` repo secret workflow
6. Define Vault `KV v2` publication paths and policies
7. Implement per-environment machine identities
8. Add Ansible bootstrap and operational workflows that consume provisioned outputs and Vault-delivered secrets

**Cross-Component Dependencies:**
- Environment-root composition depends on the trust model and remote state separation
- Vault path design depends on the per-environment identity model
- Ansible workflow design depends on OpenTofu outputs and Vault secret publication
- Secret publication workflows depend on the SOPS source-of-truth format
- Capability modules depend on the hybrid repo structure and environment assembly strategy

## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical Conflict Points Identified:**
6 major areas where AI agents could make different choices and create incompatible infrastructure patterns:
- module and environment naming
- repository structure and ownership boundaries
- secret path and file mapping
- output and inventory handoff
- validation and test placement
- operational workflow conventions

### Naming Patterns

**Infrastructure Naming Conventions:**
- Environment names use lowercase identifiers: `dev`, `prod`, and later additional environment names in lowercase
- Capability names use lowercase kebab-case when represented as directories: `proxmox`, `k3s`, `postgres`, `secrets`
- OpenTofu module directories use lowercase kebab-case
- OpenTofu variable and output names use `snake_case`
- Resource logical names in OpenTofu use `snake_case`
- Ansible role names use lowercase kebab-case or snake_case consistently within the repo; default preference is kebab-case for directory names and snake_case for variable names
- Vault paths use slash-delimited lowercase segments with environment before capability

**Examples:**
- Good: `tofu/modules/proxmox-cluster`
- Good: `tofu/environments/dev`
- Good: `vault kv put secret/terminus/dev/proxmox/...`
- Good: `output "cluster_api_endpoint"`
- Avoid: `ProxmoxCluster`, `Dev`, `clusterApiEndpoint`, `secret/Proxmox/Dev/...`

**API and Integration Naming Conventions:**
- OpenTofu outputs that are consumed by Ansible must use descriptive `snake_case` names
- Generated inventory groups should use lowercase stable names
- Secret identifiers should reflect purpose, not implementation detail

**Examples:**
- Good: `proxmox_api_endpoint`, `vault_bootstrap_token`, `hypervisor_nodes`
- Avoid: `endpoint1`, `apiThing`, `nodesList`

### Structure Patterns

**Project Organization:**
- OpenTofu owns declarative infrastructure under `tofu/`
- Ansible owns guest/bootstrap and operational workflows under `ansible/`
- Environment composition happens only in environment roots
- Reusable capability logic belongs in shared modules, not duplicated in environment roots
- Secrets source files belong in a dedicated repo-visible encrypted structure, not scattered alongside arbitrary modules
- Vault publication logic belongs in explicit automation paths, not embedded ad hoc across unrelated scripts

**Directory Ownership Rules:**
- `tofu/modules/` contains reusable capability or platform primitives
- `tofu/environments/` contains environment assembly only
- `ansible/roles/` contains reusable guest or operational roles
- `ansible/playbooks/` contains orchestration entry points
- `docs/` contains runbooks, decisions, and operator-facing references
- Optional helper scripts belong under a dedicated scripts directory rather than hidden inside module trees

**Examples:**
- Good: environment root references shared module
- Good: Ansible playbook consumes OpenTofu outputs
- Avoid: duplicating full Proxmox definitions separately in both `dev` and `prod`
- Avoid: mixing Ansible tasks inside OpenTofu directories

### Format Patterns

**State and Configuration Formats:**
- OpenTofu files use standard `.tf` segmentation by concern where helpful, but avoid excessive fragmentation
- Environment-specific secret source files use SOPS-managed YAML
- Secret files should retain semantic filenames and stable path patterns
- Vault uses `KV v2` pathing with environment and capability encoded in the path taxonomy

**Secret Mapping Rules:**
- Git-encrypted source paths and Vault destination paths must map predictably
- One secret domain should not publish into multiple unrelated Vault paths without an explicit reason
- Secret names should describe the runtime purpose

**Examples:**
- Good repo source: `secrets/dev/proxmox/api-credentials.sops.yaml`
- Good Vault target: `secret/terminus/dev/proxmox/api-credentials`
- Avoid: opaque filenames or mixed environment content in one file

**Data Exchange Formats:**
- Dates and timestamps in generated machine-readable outputs use ISO 8601 where relevant
- Structured outputs passed between tools remain machine-readable and stable
- Boolean and numeric values should remain typed, not stringified without need

### Communication Patterns

**OpenTofu to Ansible Handoff:**
- OpenTofu outputs are the primary handoff mechanism for provisioned infrastructure metadata
- Ansible should consume rendered inventory or structured outputs, not scrape state files directly
- Handoff artifacts must be deterministic in name and location

**Examples:**
- Good: generate inventory input from explicit outputs
- Avoid: parse `terraform.tfstate` or rely on ad hoc shell extraction

**Vault Publication Patterns:**
- SOPS is the source-of-truth authoring format in Git
- Vault KV v2 is the runtime delivery target
- Publish workflows must be explicit, repeatable, and environment-scoped
- Publication should support idempotent updates rather than one-off operator actions where possible

**Logging and Operational Messaging:**
- Automation logs should clearly identify environment, capability, and action
- Log lines should prefer structured, grep-friendly wording over freeform prose
- Error messages should separate operator-actionable failures from transient system noise

### Process Patterns

**Error Handling Patterns:**
- OpenTofu plan/apply failures stop the workflow immediately
- Ansible operational failures must identify whether they are bootstrap, configuration, or runtime-operation failures
- Secret publication failures must never silently continue
- Validation failures must be surfaced before apply whenever possible

**Loading and Execution Patterns:**
- Environments are planned and applied independently
- Shared modules are versioned by Git state, not copied into environment trees
- Bootstrap follows a strict order:
  1. remote state configured
  2. infrastructure provisioned
  3. secrets published or confirmed in Vault
  4. Ansible bootstrap runs
- Human bootstrap actions should be minimized and documented explicitly when required

### Enforcement Guidelines

**All AI Agents MUST:**
- Keep OpenTofu, Ansible, SOPS, and Vault responsibilities separate
- Use lowercase kebab-case for directories and `snake_case` for OpenTofu variables and outputs
- Place environment assembly only under environment roots
- Publish runtime secrets to Vault KV v2 using a stable environment-first path taxonomy
- Treat SOPS-encrypted files as the Git source of truth for repo-managed secret material
- Avoid introducing new directory patterns when an existing shared pattern already fits

**Pattern Enforcement:**
- Validate structure through code review against the architecture document
- Validate OpenTofu formatting and configuration before merge
- Validate Ansible syntax and inventory generation before merge
- Review secret path changes for consistency between SOPS source paths and Vault destination paths
- Document intentional exceptions directly in the architecture or follow-on operational docs

### Pattern Examples

**Good Examples:**
- `tofu/modules/proxmox-cluster`
- `tofu/environments/dev`
- `ansible/roles/proxmox-bootstrap`
- `secrets/dev/proxmox/api-credentials.sops.yaml`
- `secret/terminus/dev/proxmox/api-credentials`
- `output "hypervisor_node_ips"`

**Anti-Patterns:**
- Duplicating whole environment definitions instead of composing shared modules
- Mixing secret plaintext into OpenTofu variables files
- Letting Ansible define infrastructure topology that OpenTofu already owns
- Using inconsistent environment names such as `Dev`, `development`, and `dev` in parallel
- Publishing Vault secrets under ad hoc, non-hierarchical paths
- Writing automation that depends on direct parsing of local state files

## Project Structure & Boundaries

### Complete Project Directory Structure

```text
terminus.infra/
├── README.md
├── .gitignore
├── .sops.yaml
├── docs/
│   ├── architecture/
│   │   └── proxmox.md
│   ├── runbooks/
│   │   ├── bootstrap.md
│   │   ├── secret-publication.md
│   │   └── disaster-recovery.md
│   └── decisions/
│       └── adr-0001-proxmox-foundation.md
├── tofu/
│   ├── modules/
│   │   ├── proxmox-cluster/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── versions.tf
│   │   │   └── README.md
│   │   ├── proxmox-network/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   ├── proxmox-storage/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   ├── vault-bootstrap/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   └── README.md
│   │   └── consul-backend-bootstrap/
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       └── README.md
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── backend.tf
│   │   │   ├── providers.tf
│   │   │   ├── versions.tf
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   ├── outputs.tf
│   │   │   ├── dev.tfvars
│   │   │   └── README.md
│   │   └── prod/
│   │       ├── backend.tf
│   │       ├── providers.tf
│   │       ├── versions.tf
│   │       ├── main.tf
│   │       ├── variables.tf
│   │       ├── outputs.tf
│   │       ├── prod.tfvars
│   │       └── README.md
│   └── shared/
│       ├── naming.tf
│       ├── locals.tf
│       └── versions.tf
├── ansible/
│   ├── ansible.cfg
│   ├── requirements.yml
│   ├── inventories/
│   │   ├── dev/
│   │   │   ├── hosts.yml
│   │   │   └── group_vars/
│   │   │       └── all.yml
│   │   └── prod/
│   │       ├── hosts.yml
│   │       └── group_vars/
│   │           └── all.yml
│   ├── playbooks/
│   │   ├── bootstrap-proxmox.yml
│   │   ├── configure-guests.yml
│   │   ├── publish-runtime-secrets.yml
│   │   └── validate-environment.yml
│   └── roles/
│       ├── proxmox-bootstrap/
│       │   ├── tasks/
│       │   │   └── main.yml
│       │   ├── templates/
│       │   └── defaults/
│       │       └── main.yml
│       ├── vault-publish/
│       │   ├── tasks/
│       │   │   └── main.yml
│       │   └── defaults/
│       │       └── main.yml
│       └── guest-baseline/
│           ├── tasks/
│           │   └── main.yml
│           ├── handlers/
│           │   └── main.yml
│           └── defaults/
│               └── main.yml
├── secrets/
│   ├── dev/
│   │   └── proxmox/
│   │       ├── api-credentials.sops.yaml
│   │       ├── bootstrap-tokens.sops.yaml
│   │       └── guest-baseline.sops.yaml
│   └── prod/
│       └── proxmox/
│           ├── api-credentials.sops.yaml
│           ├── bootstrap-tokens.sops.yaml
│           └── guest-baseline.sops.yaml
├── scripts/
│   ├── tofu-plan.sh
│   ├── tofu-apply.sh
│   ├── render-inventory.sh
│   ├── publish-secrets.sh
│   └── validate.sh
├── tests/
│   ├── tofu/
│   │   ├── fmt.sh
│   │   └── validate.sh
│   ├── ansible/
│   │   ├── syntax.sh
│   │   └── inventory.sh
│   └── integration/
│       └── smoke-bootstrap.sh
└── .github/
    └── workflows/
        ├── validate-infra.yml
        └── publish-secrets.yml
```

### Architectural Boundaries

**Provisioning Boundary:**
- `tofu/` is the sole authority for Proxmox infrastructure lifecycle
- Proxmox topology, network, storage, and foundational platform resources are defined only through OpenTofu modules and environment roots
- Environment roots assemble modules; they do not contain duplicated infrastructure logic

**Configuration Boundary:**
- `ansible/` owns guest/bootstrap and post-provision operational workflows
- Ansible does not define source-of-truth infrastructure topology already owned by OpenTofu
- Generated inventory or structured outputs are the handoff point from OpenTofu to Ansible

**Secret Boundary:**
- `secrets/` contains SOPS-encrypted Git source artifacts only
- Vault KV v2 is the runtime destination for selected secret material
- Secret publication logic lives in dedicated playbooks or scripts, not embedded ad hoc across the repo

**State Boundary:**
- Each environment root uses its own remote Consul-backed state configuration
- State is environment-scoped and must not be shared by unrelated environments
- Agents must not depend on direct parsing of local state files outside supported output flows

### Requirements to Structure Mapping

**Feature/Capability Mapping:**
- Proxmox cluster lifecycle:
  - `tofu/modules/proxmox-cluster/`
  - `tofu/environments/dev/main.tf`
  - `tofu/environments/prod/main.tf`
- Proxmox networking:
  - `tofu/modules/proxmox-network/`
- Proxmox storage and datastore concerns:
  - `tofu/modules/proxmox-storage/`
- Guest/bootstrap operations:
  - `ansible/playbooks/bootstrap-proxmox.yml`
  - `ansible/roles/proxmox-bootstrap/`
  - `ansible/roles/guest-baseline/`
- Runtime secret publication:
  - `secrets/<env>/proxmox/*.sops.yaml`
  - `ansible/playbooks/publish-runtime-secrets.yml`
  - `ansible/roles/vault-publish/`

**Cross-Cutting Concerns:**
- Remote state:
  - `tofu/environments/*/backend.tf`
  - `tofu/modules/consul-backend-bootstrap/`
- Vault runtime integration:
  - `tofu/modules/vault-bootstrap/`
  - `ansible/roles/vault-publish/`
  - `docs/runbooks/secret-publication.md`
- Validation:
  - `tests/tofu/`
  - `tests/ansible/`
  - `scripts/validate.sh`

### Integration Points

**Internal Communication:**
- OpenTofu outputs feed inventory rendering and bootstrap inputs
- Scripts or playbooks transform environment outputs into Ansible inventory
- SOPS-managed files are decrypted only at explicit publication or execution boundaries

**External Integrations:**
- Proxmox API via `bpg/proxmox`
- Consul for OpenTofu remote state
- Vault KV v2 for runtime secret delivery
- Ansible `community.proxmox` collection for selective operational workflows where needed

**Data Flow:**
1. Environment root selects and configures shared OpenTofu modules
2. OpenTofu provisions or updates Proxmox resources
3. OpenTofu outputs are rendered into inventory or structured bootstrap inputs
4. SOPS-managed source secrets are published into Vault KV v2
5. Ansible consumes inventory plus Vault-delivered secret material for bootstrap and operations

### File Organization Patterns

**Configuration Files:**
- Root-level config files define repo-wide behavior
- Environment-specific OpenTofu config is isolated under `tofu/environments/<env>/`
- Ansible runtime config lives under `ansible/`
- SOPS repo rules are centralized in `.sops.yaml`

**Source Organization:**
- Reusable infrastructure primitives live in `tofu/modules/`
- Environment assembly lives in `tofu/environments/`
- Reusable operational logic lives in `ansible/roles/`
- Execution entry points live in `ansible/playbooks/` and `scripts/`

**Test Organization:**
- OpenTofu validation scripts live under `tests/tofu/`
- Ansible validation scripts live under `tests/ansible/`
- Higher-level bootstrap or smoke checks live under `tests/integration/`

**Asset Organization:**
- This repo does not have frontend static assets
- Documentation and operator artifacts live under `docs/`
- Generated files should remain ephemeral unless explicitly part of the repo contract

### Development Workflow Integration

**Development Server Structure:**
- Development happens per environment root, not from a monolithic shared execution directory
- Operators and agents should work from the relevant environment directory for plan/apply actions

**Build Process Structure:**
- Validation runs first on module and environment configuration
- Provisioning and publication are explicit steps, not implicit side effects
- OpenTofu, secret publication, and Ansible execution remain separate pipeline stages

**Deployment Structure:**
- Deployment is environment-scoped
- Secret publication and bootstrap happen after infrastructure convergence
- Environment isolation is maintained consistently across state, identity, inventory, and Vault paths

## Architecture Validation Results

### Coherence Validation ✅

**Decision Compatibility:**
The selected technologies and control boundaries are compatible and mutually reinforcing.

- OpenTofu with `bpg/proxmox` provides a clear provisioning authority for Proxmox-managed infrastructure
- Consul remote state aligns with the requirement for team-safe shared state and environment-level separation
- SOPS with `age` provides a Git-native secret authoring flow that does not conflict with OpenTofu or Ansible responsibilities
- Vault `KV v2` cleanly separates runtime secret delivery from source-controlled secret authoring
- Ansible remains appropriately scoped to bootstrap and operational automation rather than competing with infrastructure provisioning
- The hybrid project structure supports both reusable capability modules and environment-root composition without structural contradiction

**Pattern Consistency:**
The implementation patterns support the architecture decisions and reduce likely multi-agent conflicts.

- Naming rules are consistent across modules, environments, OpenTofu outputs, and Vault paths
- Structure rules preserve clear authority boundaries between `tofu/`, `ansible/`, `secrets/`, and `docs/`
- Communication and handoff patterns consistently use OpenTofu outputs rather than ad hoc state parsing
- Secret handling patterns preserve the distinction between Git-encrypted source material and runtime secret publication

**Structure Alignment:**
The project structure supports the chosen architecture well.

- Environment roots map cleanly to the per-environment trust and state model
- Shared modules support the hybrid composition model
- Secret source and runtime secret publication are separated structurally and operationally
- Validation, documentation, and operational workflows have defined homes in the repository

### Requirements Coverage Validation ✅

**Feature Coverage:**
The current requirements for a Proxmox-focused infrastructure initiative are supported by the architecture.

- Proxmox cluster lifecycle is covered through dedicated OpenTofu modules and environment assembly
- Networking, storage, and bootstrap concerns have defined architectural homes
- Secret publication and runtime delivery concerns are explicitly supported
- Operational and bootstrap automation needs are covered through the Ansible layer

**Functional Requirements Coverage:**
The architecture addresses the core functional expectations implied by the technical requirements.

- It supports infrastructure definition and standardization at the Proxmox layer
- It defines where integrations, automation, and operational workflows should live
- It establishes implementation boundaries sufficient for later sprint planning and execution

**Non-Functional Requirements Coverage:**
The architecture addresses the main non-functional concerns raised earlier.

- Security is addressed through environment-separated identities, encrypted repo secrets, and Vault-delivered runtime secrets
- Operational clarity is addressed through role separation and explicit workflow boundaries
- Drift control is addressed by making OpenTofu the provisioning authority and Consul the remote state backend
- Recovery and maintainability are supported by explicit runbook and decision-document locations
- Scalability is handled structurally through reusable modules and environment-root composition

### Implementation Readiness Validation ✅

**Decision Completeness:**
Critical architectural choices are documented clearly enough to guide implementation.

- Provisioning authority is defined
- State backend is defined
- Secret authoring and runtime delivery models are defined
- Environment trust boundaries are defined
- Repo organization and implementation patterns are defined

**Structure Completeness:**
The proposed project structure is specific enough for agents to start implementation consistently.

- Root directories and major files are defined
- OpenTofu, Ansible, secret, and documentation boundaries are mapped
- Integration points and workflow handoffs are explicit
- Validation and operational support paths are included

**Pattern Completeness:**
The implementation rules are sufficient to reduce avoidable conflicts between agents.

- Naming conventions are explicit
- Directory ownership is explicit
- Handoff patterns are explicit
- Secret path and publication patterns are explicit
- Error and execution sequencing expectations are explicit

### Gap Analysis Results

**Critical Gaps:**
- None identified at the current planning level

**Important Gaps:**
- Exact Proxmox topology is deferred
- Exact Vault path taxonomy is deferred
- Exact Consul state key layout is deferred
- Exact CI/CD realization is deferred

These do not block architecture completion, but they should be resolved before or during early implementation stories.

**Nice-to-Have Gaps:**
- More detailed observability standards
- More detailed backup and recovery mechanics
- Additional examples for module composition and inventory rendering
- Explicit ADRs for secret publication and state conventions

### Validation Issues Addressed

No blocking contradictions were found in the architecture.

The primary issues discovered during validation were incompleteness risks rather than errors:
- risk of agent drift in repo structure
- risk of mixed infrastructure/configuration authority
- risk of unclear secret publication boundaries

These were addressed by the implementation-pattern rules and the explicit project structure and boundary definitions.

### Architecture Completeness Checklist

**✅ Requirements Analysis**
- [x] Project context thoroughly analyzed
- [x] Scale and complexity assessed
- [x] Technical constraints identified
- [x] Cross-cutting concerns mapped

**✅ Architectural Decisions**
- [x] Critical decisions documented with versions
- [x] Technology stack fully specified
- [x] Integration patterns defined
- [x] Performance and operational considerations addressed at the architecture level

**✅ Implementation Patterns**
- [x] Naming conventions established
- [x] Structure patterns defined
- [x] Communication patterns specified
- [x] Process patterns documented

**✅ Project Structure**
- [x] Complete directory structure defined
- [x] Component boundaries established
- [x] Integration points mapped
- [x] Requirements to structure mapping complete

### Architecture Readiness Assessment

**Overall Status:** READY FOR IMPLEMENTATION

**Confidence Level:** High

**Key Strengths:**
- Clear control-plane ownership boundaries
- Strong separation of provisioning, configuration, state, and secrets
- Consistent multi-environment model
- Practical Git-native secret workflow
- Structure and pattern rules that reduce agent inconsistency

**Areas for Future Enhancement:**
- Formalize Vault path taxonomy
- Formalize Consul state naming and backend conventions
- Define observability and backup standards in more detail
- Expand ADR coverage for operational practices

### Implementation Handoff

**AI Agent Guidelines:**
- Follow architectural boundaries exactly as documented
- Use the implementation patterns as mandatory defaults
- Keep OpenTofu, Ansible, SOPS, and Vault concerns separate
- Treat environment roots as composition points, not duplication points
- Use this document as the authority for repo structure and integration behavior

**First Implementation Priority:**
Bootstrap the repository skeleton and control-plane foundation:
1. establish `tofu/`, `ansible/`, `secrets/`, `docs/`, `scripts/`, and `tests/`
2. define Consul-backed environment roots
3. wire in `bpg/proxmox`
4. establish `.sops.yaml` and initial encrypted secret layout
5. add Vault KV v2 publication workflow