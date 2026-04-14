# Story 3.1: Create Encrypted Secret Source Layout

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want environment-scoped SOPS-encrypted secret source files for Proxmox automation,
so that sensitive values are versioned safely and remain separated by environment.

## Acceptance Criteria

1. `secrets/dev/proxmox/` and `secrets/prod/proxmox/` contain the approved encrypted file skeletons.
2. Secret filenames are semantic and match the taxonomy defined in Story 1.2 and Story 1.3.
3. No plaintext secrets are present in tracked files created or modified by this story.
4. Validation catches malformed or missing encrypted secret files in the approved locations.
5. The secret source layout is sufficient to support Vault publication in the next story without renaming.

## Tasks / Subtasks

- [ ] Write failing validation checks FIRST for encrypted secret file placement, naming, and absence of plaintext secrets.
- [ ] Create the approved encrypted secret skeletons for `dev` and `prod` under the canonical path structure.
- [ ] Verify SOPS-managed file metadata aligns with `.sops.yaml` and the approved secret taxonomy.
- [ ] Confirm no plaintext secret files, decrypted artifacts, or ad hoc secret paths are introduced.
- [ ] Document the expected file set so Story 3.2 can publish to Vault without renaming or remapping sources.

## Dev Notes

- Story 3.1 creates encrypted source-of-truth material only. It does not publish runtime values into Vault.
- Epic 3 readiness requires stable filenames and directories so Story 3.2 can map them without special cases or exceptions.
- Follow the `.sops.yaml` and `age` custody conventions defined in Story 1.3 exactly; do not invent alternate secret-management rules here.
- Secret filenames should remain semantic and environment-scoped rather than embedding incidental implementation details.
- Validation should reject missing encrypted files, malformed locations, or anything that looks like plaintext source material.

### Project Structure Notes

- This story should primarily affect `secrets/dev/proxmox/`, `secrets/prod/proxmox/`, `.sops.yaml`, and validation under `tests/` plus `scripts/validate.sh`.
- Keep source secret layout distinct from Vault runtime paths; Git source files and Vault destinations are separate authorities.
- The file set created here must be directly consumable by the Story 3.2 publication workflow.
- Avoid adding publication logic, Vault policies, or runtime-only metadata into the encrypted source layout.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-31-Create-Encrypted-Secret-Source-Layout]
- Epic 3 readiness note on stable names and directories: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-3.md#Findings]
- Technical decision for SOPS-managed source files: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-005-Use-SOPS-for-Repo-Managed-Secret-Source-Files]
- Technical decision for `age` key management: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-006-Use-age-for-SOPS-Key-Management]
- Architecture secret boundary and source/destination separation: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture example secret file layout: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 3 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Stable secret-layout and no-plaintext constraints made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/3-1-create-encrypted-secret-source-layout.md
