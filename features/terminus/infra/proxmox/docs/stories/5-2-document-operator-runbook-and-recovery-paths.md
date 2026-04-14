# Story 5.2: Document Operator Runbook And Recovery Paths

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a returning operator,
I want a complete runbook for setup, execution, recovery, and repeated operation,
so that I can restore service or continue planned work without needing source-level tribal knowledge.

## Acceptance Criteria

1. The runbook documents initial setup, validation, provisioning flow, secret publication flow, bootstrap flow, and common failure recovery.
2. The runbook explains the required operator-managed steps for `age` key custody and Vault access.
3. The runbook includes the standard commands for `dev` environment bootstrap and validation.
4. The runbook identifies where logs, outputs, and inventory artifacts should be inspected during troubleshooting.
5. A reader can follow the runbook without consulting implementation files to understand the expected operating sequence.

## Tasks / Subtasks

- [ ] Write the runbook skeleton FIRST with placeholders for each required operating stage.
- [ ] Document setup, validation, provisioning, secret publication, and bootstrap steps.
- [ ] Add recovery guidance for backend, Vault, inventory, and bootstrap failures.
- [ ] Include the standard `dev` bootstrap and validation command paths.
- [ ] Verify the runbook is understandable without implementation-code context.

## Dev Notes

- This story captures operator readiness explicitly. It should be readable by someone returning cold to the repo.
- Epic 5 readiness requires documenting the real operator-managed steps for Vault access and `age` key custody without burying them in implementation details.
- The runbook should reflect the actual execution order established in architecture and prior stories, not an idealized or parallelized sequence.
- Recovery guidance should classify failures by backend, provisioning, secret publication, inventory, and bootstrap stages.
- Keep the runbook operator-facing and procedural rather than turning it into a code walk-through.

### Project Structure Notes

- This story will likely affect `docs/runbooks/` and any cross-references to validation, inventory, or evidence locations created by earlier stories.
- The architecture starter layout already reserves `docs/runbooks/bootstrap.md`, `docs/runbooks/secret-publication.md`, and `docs/runbooks/disaster-recovery.md` as likely homes or anchors.
- Keep command examples environment-scoped and make the standard `dev` path explicit for repeatability.
- The runbook should point to stable artifact and evidence locations rather than one-off terminal transcripts.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-52-Document-Operator-Runbook-And-Recovery-Paths]
- Epic 5 readiness note on real operator-managed guidance: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-5.md#Findings]
- Technical decisions deferred around observability, backup, and recovery specifics: [Source: docs/terminus/infra/proxmox/tech-decisions.md#Deferred-Decisions]
- Architecture documentation and runbook structure: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]
- Architecture recovery and maintainability rationale: [Source: docs/terminus/infra/proxmox/architecture.md#Requirements-Coverage-Validation]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 5 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Operator-facing recovery and custody guidance expectations made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md
