# Story 1.2: Lock Consul State And Vault Path Conventions

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the Consul state key naming convention and Vault KV v2 path taxonomy documented in a committed decision record,
so that every environment and automation path uses a single predictable convention.

## Acceptance Criteria

1. A decision document defines the Consul backend key pattern for all environments.
2. A decision document defines the environment-first Vault KV v2 path taxonomy for all Proxmox secret material.
3. The documented conventions match the architecture constraints and do not conflict with existing terminus governance.
4. Story 2 and Story 3 implementation inputs can reference the documented patterns without ambiguity.
5. The decision record includes at least one concrete example for both `dev` and `prod`.

## Tasks / Subtasks

- [ ] Add failing-first validation or lint coverage that asserts the required Consul key and Vault path examples are present in the decision record.
- [ ] Create the initial decision record under `docs/decisions/` in `terminus.infra` for the shared conventions established by this story.
- [ ] Define the canonical Consul backend key pattern for environment-scoped OpenTofu state and include concrete `dev` and `prod` examples.
- [ ] Define the canonical Vault KV v2 environment-first path taxonomy for Proxmox secret publication and include concrete `dev` and `prod` examples.
- [ ] Cross-check the decision record against architecture and technical decisions so later stories can consume the conventions without reinterpretation.

## Dev Notes

- Story 1.2 is decision-focused. Do not implement actual Consul backend blocks, Vault policies, publication scripts, or secret files here.
- This story closes two explicitly deferred decisions: exact Consul state key naming and exact Vault path taxonomy. Those conventions are a hard prerequisite for Epics 2 and 3.
- Story 1.1 has already established the baseline repository skeleton in `terminus.infra`; this story should build on that by adding documentation only, not structural scaffolding.
- Keep the conventions environment-scoped and deterministic. The architecture requires each environment root to own separate remote state, and Vault paths must remain environment-first.
- The target decision record should live in the target repo under `docs/decisions/`, since the tech decisions log says future implementation exceptions and concrete ADRs belong there.
- Prefer concise, implementation-ready naming rules that downstream stories can copy directly into `backend.tf`, publication scripts, and secret path definitions.

### Project Structure Notes

- The target repository remains the shared `terminus.infra` service repo.
- Story 1.2 should add documentation under a path like `docs/decisions/adr-0001-proxmox-state-and-vault-conventions.md`.
- The conventions documented here must be usable by later paths such as `tofu/environments/dev/backend.tf`, `tofu/environments/prod/backend.tf`, `secrets/dev/proxmox/`, `secrets/prod/proxmox/`, and future Vault publication workflows under `ansible/` or `scripts/`.
- Keep naming aligned with the existing repo rules: lowercase environment names, stable capability names, and predictable path segments.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-12-Lock-Consul-State-And-Vault-Path-Conventions]
- Hard prerequisite note for later epics: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-1.md#Findings]
- Deferred decisions to close in this story: [Source: docs/terminus/infra/proxmox/tech-decisions.md#Deferred-Decisions]
- Consul remote-state decision and constraints: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-003-Store-Remote-State-in-Consul]
- Vault runtime-secret decision and example taxonomy: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-007-Use-Vault-KV-v2-for-Runtime-Secret-Delivery]
- Architecture constraints for environment-scoped state and environment-first Vault paths: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture implementation roadmap calling for formalized conventions: [Source: docs/terminus/infra/proxmox/architecture.md#Implementation-Readiness-Validation]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story 1.1 implementation merged in `terminus.infra` PR #1.
- Sprint status advanced after verifying the merged story PR.

### Completion Notes List

- Story 1.1 marked `done` after merge.
- Story 1.2 promoted to `ready-for-dev` as the next handoff in Epic 1.
- Story 1.2 constrained to decision-record work only so Epics 2 and 3 can consume stable conventions.

### File List

- docs/terminus/infra/proxmox/sprint-status.yaml
- docs/terminus/infra/proxmox/stories/1-2-lock-consul-state-and-vault-path-conventions.md