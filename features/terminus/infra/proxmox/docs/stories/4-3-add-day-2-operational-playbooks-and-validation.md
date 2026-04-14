# Story 4.3: Add Day-2 Operational Playbooks And Validation

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want day-2 operational playbooks for validation and repeatable maintenance actions,
so that the infra service has a supported operational workflow after initial bootstrap.

## Acceptance Criteria

1. At least one validation-oriented playbook exists for environment verification after bootstrap.
2. Operational playbooks consume the same inventory and secret patterns as bootstrap.
3. Operational actions are scoped to infra-service concerns and do not duplicate OpenTofu provisioning logic.
4. Playbook failure modes are documented with actionable operator guidance.
5. Validation and syntax checks are present for each added operational playbook.

## Tasks / Subtasks

- [ ] Write failing validation checks FIRST for the new operational playbooks.
- [ ] Implement at least one validation-oriented day-2 playbook.
- [ ] Reuse the same inventory and secret consumption patterns as the bootstrap path.
- [ ] Document actionable failure modes for operators.
- [ ] Verify syntax and validation coverage for each new playbook.

## Dev Notes

- This story is for supported operational workflows after bootstrap, not a second provisioning system. Keep the scope aligned with infra ownership.
- Epic 4 readiness explicitly warns against letting day-2 operations become a shadow provisioning path.
- Reuse the same inventory and secret-delivery conventions established in Stories 4.1 and 4.2 so operators do not have to learn separate workflows.
- Failure modes should classify whether the problem is bootstrap, configuration, or runtime operation, matching the architecture guidance.
- Keep these playbooks focused on validation and maintenance actions that belong to the infra service boundary.

### Project Structure Notes

- This story will primarily affect `ansible/playbooks/`, `tests/ansible/`, and operator-facing documentation under `docs/`.
- Operational playbooks should sit alongside the bootstrap path and share the same inventory and Vault-delivered secret assumptions.
- Keep the execution model Ansible-centric and avoid coupling operational workflows back into OpenTofu provisioning logic.
- Actionable failure guidance created here should feed forward into the later operator runbook.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-43-Add-Day-2-Operational-Playbooks-And-Validation]
- Epic 4 readiness note on avoiding a second provisioning path: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-4.md#Findings]
- Technical decision on Ansible ownership boundary: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-004-Keep-Configuration-Management-in-Ansible]
- Architecture operations and failure-mode guidance: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture validation and Ansible test placement: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 4 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Day-2 operational boundaries and failure-mode expectations made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/4-3-add-day-2-operational-playbooks-and-validation.md
