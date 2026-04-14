# Story 2.2: Implement Proxmox Module Skeleton And Environment Composition

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want reusable OpenTofu modules and environment composition for Proxmox cluster, network, and storage concerns,
so that infrastructure can be planned and applied declaratively without duplicating topology definitions per environment.

## Acceptance Criteria

1. Shared modules exist for the agreed Proxmox capabilities needed by the baseline architecture.
2. Environment roots compose shared modules rather than duplicating complete infrastructure definitions inline.
3. The `bpg/proxmox` provider is configured in a way compatible with the approved architecture decisions.
4. Module inputs and outputs use `snake_case` naming consistently.
5. Validation coverage exists for module structure and basic variable contracts before implementation is finalized.

## Tasks / Subtasks

- [ ] Write failing module validation checks FIRST for required files, module structure, and baseline variable contracts.
- [ ] Create shared modules for the baseline Proxmox capabilities named in the architecture.
- [ ] Wire environment roots to those shared modules rather than duplicating topology definitions inline.
- [ ] Configure the `bpg/proxmox` provider consistently with the approved architecture and tech decisions.
- [ ] Verify module naming, structure, and output conventions remain consistent with repository rules.

## Dev Notes

- Story 2.2 is about module skeleton and composition, not final production topology. Epic 2 readiness explicitly warns against absorbing deferred topology decisions here.
- Favor the architecture’s baseline module set and composition pattern over ad hoc environment-specific definitions.
- Keep provider usage and module I/O naming consistent with `snake_case` conventions because later inventory and bootstrap stories depend on stable outputs.
- Modules should remain reusable primitives, while the environment roots remain the only place where environment-specific assembly occurs.
- Validation should catch missing module files, malformed variable declarations, or duplicated inline topology before merge.

### Project Structure Notes

- This story should primarily affect `tofu/modules/`, `tofu/environments/dev/`, `tofu/environments/prod/`, and supporting validation under `tests/tofu/`.
- The architecture’s starter layout includes modules for cluster, network, storage, vault bootstrap, and Consul backend bootstrap; only implement the baseline skeleton required for this story.
- Keep module interfaces narrow and consistent so Story 2.3 can expose deterministic outputs without reworking module contracts.
- Avoid scattering provider or topology logic into unrelated script or Ansible paths.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-22-Implement-Proxmox-Module-Skeleton-And-Environment-Composition]
- Epic 2 readiness note on unresolved topology scope: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-2.md#Findings]
- OpenTofu provisioning authority decision: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-001-Use-OpenTofu-as-the-Provisioning-Authority]
- Proxmox provider decision: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-002-Use-bpgproxmox-for-Proxmox-Control]
- Hybrid composition decision: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-009-Use-a-Hybrid-Repository-Composition-Model]
- Architecture project structure and module layout: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 2 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Module-skeleton scope and deferred-topology guardrails made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/2-2-implement-proxmox-module-skeleton-and-environment-composition.md
