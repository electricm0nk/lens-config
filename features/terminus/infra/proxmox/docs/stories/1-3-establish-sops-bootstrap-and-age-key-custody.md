# Story 1.3: Establish SOPS Bootstrap And Age Key Custody

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want `.sops.yaml`, the `age` recipient model, and operator key-custody steps defined,
so that encrypted secrets can be created safely without ad hoc manual handling.

## Acceptance Criteria

1. `.sops.yaml` exists at the repository root and maps the approved secret file locations.
2. The operator bootstrap document defines how `age` keys are created, stored, and shared with authorized automation and humans.
3. An example encrypted secret file path is defined for both `dev` and `prod`.
4. The custody procedure makes clear that plaintext secrets are never committed to Git.
5. A validation check fails when an expected secret file is unencrypted or placed outside the approved path pattern.

## Tasks / Subtasks

- [ ] Write failing validation checks FIRST for `.sops.yaml` presence, approved secret paths, and rejection of unencrypted secret files.
- [ ] Add `.sops.yaml` at the repository root with rules that cover `secrets/dev/proxmox/` and `secrets/prod/proxmox/`.
- [ ] Document `age` key creation, storage, operator custody, and automation access as distinct responsibilities.
- [ ] Add example encrypted secret file targets for both environments without committing plaintext or real secret values.
- [ ] Verify validation fails when secret files are unencrypted, misplaced, or outside the approved path pattern.

## Dev Notes

- Story 1.3 is the control-plane secret bootstrap story. It defines encryption rules and custody guidance, but it does not publish runtime values to Vault yet.
- Keep human key custody and automation access documented separately. Epic 1 readiness explicitly warns against blurring those two responsibility models.
- This story depends on Story 1.2 conventions, especially the approved environment-first naming and path rules that later secret publication will consume.
- Do not commit plaintext secret values, test fixtures with real credentials, or ad hoc temporary decrypted files.
- The outcome must be stable enough that Epic 3 can add encrypted secret files and Vault publication without renaming paths or reworking custody guidance.

### Project Structure Notes

- The target repository is still the shared `terminus.infra` service repository.
- Story 1.3 should primarily affect paths such as `.sops.yaml`, `secrets/dev/proxmox/`, `secrets/prod/proxmox/`, and a bootstrap or runbook document under `docs/`.
- Keep filenames semantic and environment-scoped so Story 3.1 can create encrypted source files without introducing special-case naming.
- Validation for this story should stay lightweight and structural, using repo checks under `tests/` and the main validation entry point under `scripts/validate.sh`.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-13-Establish-SOPS-Bootstrap-And-Age-Key-Custody]
- Epic 1 readiness note on separating operator and automation custody: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-1.md#Findings]
- Technical decision for SOPS-managed source files: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-005-Use-SOPS-for-Repo-Managed-Secret-Source-Files]
- Technical decision for `age` key management: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-006-Use-age-for-SOPS-Key-Management]
- Architecture secret boundary and source layout: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture file organization and secret examples: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 1 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Custody separation and no-plaintext constraints made explicit for downstream implementation.

### File List

- docs/terminus/infra/proxmox/stories/1-3-establish-sops-bootstrap-and-age-key-custody.md
