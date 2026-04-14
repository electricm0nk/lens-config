# Story 5.1: Build End-To-End Validation Harness

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want a single validation entry point that checks the full bootstrap chain,
so that I can prove the environment is ready without manually stitching together each tool invocation.

## Acceptance Criteria

1. A single validation entry point verifies backend readiness, provisioning prerequisites, secret publication prerequisites, inventory generation, and bootstrap readiness.
2. Validation output identifies the failing stage clearly when the chain is broken.
3. The harness does not mutate production infrastructure as part of validation-only execution.
4. Tests for the validation harness are written before final implementation changes for this story.
5. The validation entry point is documented as the standard readiness check for future operators.

## Tasks / Subtasks

- [ ] Write failing tests FIRST for the stage-by-stage validation harness behavior.
- [ ] Implement a single validation entry point that checks each stage in order.
- [ ] Ensure the harness distinguishes readiness checks from mutating operations.
- [ ] Surface stage-specific failure output for operator diagnosis.
- [ ] Document the validation harness as the standard readiness command.

## Dev Notes

- The validation harness is a control-plane check, not a replacement for provisioning or bootstrap execution. Keep side effects out of validation mode.
- Epic 5 readiness explicitly requires non-mutating behavior. Treat validation-only execution as a hard safety boundary.
- The harness should stitch together existing backend, OpenTofu, secret publication, inventory, and bootstrap checks rather than creating alternate workflows.
- Stage-specific diagnostics are part of the contract because later smoke validation and runbooks will rely on them.
- Keep the validation command stable and operator-oriented so it can become the default readiness entry point.

### Project Structure Notes

- This story will likely affect `scripts/validate.sh`, `tests/tofu/`, `tests/ansible/`, `tests/integration/`, and operator-facing documentation under `docs/`.
- Keep validation composition explicit and ordered, matching the architecture’s bootstrap sequence.
- Do not let the validation harness call mutating apply or bootstrap actions when operating in validation mode.
- Higher-level smoke and readiness checks should remain compatible with the standard harness introduced here.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-51-Build-End-To-End-Validation-Harness]
- Epic 5 readiness note on non-mutating validation: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-5.md#Findings]
- Architecture validation sequencing and test placement: [Source: docs/terminus/infra/proxmox/architecture.md#Development-Workflow-Integration]
- Architecture validation/test organization: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]
- Architecture bootstrap order: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 5 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Non-mutating validation-mode and stage-diagnostic requirements made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/5-1-build-end-to-end-validation-harness.md
