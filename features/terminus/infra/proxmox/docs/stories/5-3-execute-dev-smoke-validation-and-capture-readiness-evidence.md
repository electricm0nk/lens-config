# Story 5.3: Execute Dev Smoke Validation And Capture Readiness Evidence

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want a documented dev smoke validation run,
so that the full chain is proven once before later sprint planning or execution work depends on it.

## Acceptance Criteria

1. The `dev` environment smoke flow executes the approved validation sequence end to end.
2. Evidence of success or failure is captured in a repeatable location referenced by the runbook.
3. Any unresolved failure is classified by stage: backend, provisioning, secret publication, inventory, or bootstrap.
4. The smoke validation does not require undocumented manual steps.
5. The resulting evidence is sufficient to support implementation-readiness review for the initiative.

## Tasks / Subtasks

- [ ] Write the smoke validation checklist FIRST.
- [ ] Execute the approved `dev` validation sequence and capture results.
- [ ] Record success or failure evidence in the agreed repeatable location.
- [ ] Classify any failure by stage with actionable notes.
- [ ] Verify the runbook reflects the actual executed sequence with no undocumented steps.

## Dev Notes

- This story is the proof point that the earlier epics converge into an operator-usable path. Keep the evidence location stable and explicit.
- Epic 5 readiness explicitly requires a stable evidence location referenced by the runbook. Avoid ad hoc screenshots, terminal-only notes, or ephemeral output paths.
- The smoke flow should use the standard validation and execution commands established earlier rather than special one-off commands.
- Failure classification must match the earlier stage boundaries so readiness evidence is actionable, not just archival.
- Keep the scope to `dev`; this is readiness proof, not a full production certification story.

### Project Structure Notes

- This story will likely affect `tests/integration/`, runbook references under `docs/runbooks/`, and an agreed evidence location under `docs/` or another stable repository path.
- The evidence location should be repeatable and operator-discoverable, not tied to a single local machine or session transcript.
- Keep the smoke sequence aligned with the validation harness and runbook introduced in Stories 5.1 and 5.2.
- Any captured evidence should be stage-classified and suitable for later readiness review.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-53-Execute-Dev-Smoke-Validation-And-Capture-Readiness-Evidence]
- Epic 5 readiness note on stable evidence location: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-5.md#Findings]
- Architecture test and smoke-check placement: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]
- Architecture execution order and readiness validation flow: [Source: docs/terminus/infra/proxmox/architecture.md#Development-Workflow-Integration]
- Architecture recovery and maintainability rationale: [Source: docs/terminus/infra/proxmox/architecture.md#Requirements-Coverage-Validation]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 5 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Stable-evidence and stage-classification requirements made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/5-3-execute-dev-smoke-validation-and-capture-readiness-evidence.md
