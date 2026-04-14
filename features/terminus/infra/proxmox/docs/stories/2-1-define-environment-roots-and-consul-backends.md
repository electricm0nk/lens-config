# Story 2.1: Define Environment Roots And Consul Backends

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want `dev` and `prod` OpenTofu environment roots with remote Consul backend configuration,
so that infrastructure state is shared safely and isolated per environment.

## Acceptance Criteria

1. `tofu/environments/dev/` and `tofu/environments/prod/` each contain backend, provider, version, main, variable, and output files.
2. Each environment root uses the approved Consul key naming convention from Story 1.2.
3. Local state files are excluded from source control and not required for normal operation.
4. Validation commands succeed for both environment roots with no backend naming ambiguity.
5. Tests or validation scripts for backend configuration are written before the final implementation for this story.

## Tasks / Subtasks

- [ ] Write failing backend validation checks FIRST for both environment roots and the approved Consul key naming convention.
- [ ] Create the `dev` and `prod` environment root file sets under `tofu/environments/` with the expected baseline files.
- [ ] Configure Consul backend blocks using the conventions locked in Story 1.2.
- [ ] Ensure local state artifacts are ignored, unnecessary for normal operation, and not consumed by helper scripts.
- [ ] Verify validation succeeds for both environments and that backend naming remains deterministic and unambiguous.

## Dev Notes

- This story is the first real OpenTofu implementation step after the decision prerequisites. Keep it focused on environment-root composition and remote-state isolation.
- Epic 2 readiness is explicit: environment-scoped backend keys must follow Story 1.2 exactly. Any drift here will cascade into later validation and bootstrap issues.
- Do not widen scope into module composition, provider topology breadth, or downstream inventory/output work. Those belong to Stories 2.2 and 2.3.
- The environment roots should establish the file contract expected by the architecture, not encode unresolved production topology decisions.
- Validation should cover backend presence, naming, and environment separation before any final implementation is considered complete.

### Project Structure Notes

- This story should primarily affect `tofu/environments/dev/`, `tofu/environments/prod/`, and validation paths under `tests/tofu/` plus `scripts/validate.sh`.
- Keep backend configuration environment-scoped and consistent with the architecture starter layout for each root.
- Do not introduce local-state workflow dependencies, raw tfstate parsing, or environment-crossing shared backend keys.
- The canonical backend naming source is Story 1.2, now stored under `docs/terminus/infra/proxmox/stories/`.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-21-Define-Environment-Roots-And-Consul-Backends]
- Epic 2 readiness note on backend-key drift: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-2.md#Findings]
- Consul backend technical decision: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-003-Store-Remote-State-in-Consul]
- Environment-root execution order: [Source: docs/terminus/infra/proxmox/tech-decisions.md#Initial-Execution-Order]
- Architecture state boundary and environment isolation: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture project structure for environment roots: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 2 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Backend-key exactness and environment-root constraints made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/2-1-define-environment-roots-and-consul-backends.md
