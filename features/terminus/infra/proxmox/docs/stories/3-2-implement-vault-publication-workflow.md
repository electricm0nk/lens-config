# Story 3.2: Implement Vault Publication Workflow

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want a repeatable publish workflow that maps encrypted source files into Vault KV v2 paths,
so that runtime consumers receive the required secret material from Vault rather than from Git decryption at execution time.

## Acceptance Criteria

1. A dedicated publication playbook or script reads approved encrypted inputs and writes to Vault KV v2 using the documented taxonomy.
2. Source-to-destination mapping is explicit and environment-scoped.
3. Publication failures stop the workflow and surface actionable errors.
4. The workflow does not write secrets to logs, temp files, or source-controlled artifacts.
5. Tests or validation checks exist for path mapping and failure handling before final implementation is completed.

## Tasks / Subtasks

- [ ] Write failing tests FIRST for Vault path mapping, failure handling, and redaction behavior.
- [ ] Implement the publication playbook or script using the approved environment-first Vault taxonomy.
- [ ] Add explicit source-to-destination mapping for each published secret group.
- [ ] Ensure failure paths stop execution cleanly and never emit secret-shaped values to logs or temporary files.
- [ ] Verify the workflow leaves no secret residue in logs, temp files, or generated artifacts.

## Dev Notes

- Keep Vault publication explicit and environment-scoped. Do not blur Git-encrypted source management with runtime secret delivery.
- Epic 3 readiness requires clean failure behavior and no secret-shaped values in logs or temporary files. Treat that as a hard implementation constraint.
- The workflow should consume the stable secret layout from Story 3.1 and the taxonomy defined in Story 1.2 without introducing exceptions.
- Publication belongs in a dedicated script or playbook, not spread ad hoc across unrelated automation paths.
- This story should focus on publication mechanics and safety, not on proving idempotency. That is the job of Story 3.3.

### Project Structure Notes

- This story will likely affect `ansible/playbooks/`, `ansible/roles/`, `scripts/`, and validation or integration checks under `tests/`.
- Vault KV v2 is the runtime destination; encrypted secret files under `secrets/` remain the Git source of truth.
- Keep the mapping explicit and environment-scoped so operators can trace which source file publishes to which Vault path.
- Avoid secret handling through ad hoc temp files or debug logging in shell or Ansible tasks.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-32-Implement-Vault-Publication-Workflow]
- Epic 3 readiness note on clean failure and no secret leakage: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-3.md#Findings]
- Technical decision for Vault runtime delivery: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-007-Use-Vault-KV-v2-for-Runtime-Secret-Delivery]
- Technical decision for environment-separated identities: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-008-Use-Per-Environment-Machine-Identities]
- Architecture secret boundary and Vault path rules: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture data flow from encrypted source to Vault publication: [Source: docs/terminus/infra/proxmox/architecture.md#Integration-Points]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 3 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Publication safety and no-secret-leakage constraints made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/3-2-implement-vault-publication-workflow.md
