# Story 3.3: Verify Idempotent Secret Publication In Dev

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the Vault publication workflow to be safe to run repeatedly in `dev`,
so that routine updates do not introduce drift or duplicate manual repair steps.

## Acceptance Criteria

1. Re-running the publish workflow in `dev` updates secrets deterministically without destructive side effects.
2. The workflow reports which environment and capability it is targeting in structured, operator-readable output.
3. Validation proves that repeated runs do not require changes when inputs are unchanged.
4. Error handling distinguishes missing source secrets from Vault connectivity or authorization failures.
5. The story records the expected operator command path for future reuse in the runbook.

## Tasks / Subtasks

- [ ] Write failing idempotency checks FIRST for repeated publication behavior in `dev`.
- [ ] Run the publication workflow against unchanged `dev` inputs and capture the expected no-op or stable-update behavior.
- [ ] Improve operator-readable output so environment and capability targets are always explicit.
- [ ] Verify error classes distinguish input, connectivity, and authorization failures.
- [ ] Record the standard operator command path for reuse in the runbook and later smoke validation.

## Dev Notes

- Use `dev` only for this validation story. Epic 3 readiness explicitly says not to widen this into a multi-environment rollout.
- The point of this story is repeatability and safe operator reuse, not new publication features.
- Repeated runs with unchanged inputs should prove stable behavior and clear operator messaging.
- Keep evidence of the standard command path and expected output shape because Epic 5 runbooks and smoke validation will depend on it.
- Treat failure-class separation as part of the contract, not a cosmetic improvement.

### Project Structure Notes

- This story will likely affect the Vault publication workflow created in Story 3.2, validation under `tests/integration/` or `tests/ansible/`, and operator-facing documentation under `docs/`.
- Keep the implementation centered on the `dev` environment path and avoid broadening file layout or inventory expectations beyond that scope.
- Any evidence or reusable command examples documented here should align with the later operator runbook.
- Do not introduce ad hoc scripts that bypass the canonical publication path just to demonstrate idempotency.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-33-Verify-Idempotent-Secret-Publication-In-Dev]
- Epic 3 readiness note on limiting scope to `dev`: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-3.md#Findings]
- Technical decision for Vault runtime delivery: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-007-Use-Vault-KV-v2-for-Runtime-Secret-Delivery]
- Architecture validation and operational flow ordering: [Source: docs/terminus/infra/proxmox/architecture.md#Development-Workflow-Integration]
- Architecture higher-level smoke test placement: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 3 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- `Dev`-only idempotency and operator-output requirements made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/3-3-verify-idempotent-secret-publication-in-dev.md
