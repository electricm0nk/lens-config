# Story 4.2: Implement Baseline Guest Bootstrap Playbook

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want an Ansible bootstrap playbook and reusable roles for baseline guest setup,
so that newly provisioned infrastructure can be brought into an operational baseline with approved secret delivery.

## Acceptance Criteria

1. A bootstrap playbook exists and uses the generated inventory from Story 4.1.
2. The playbook consumes Vault-delivered secrets rather than plaintext repo values.
3. Bootstrap responsibilities remain within Ansible and do not redefine infrastructure topology already owned by OpenTofu.
4. Playbook syntax and inventory validation pass before the implementation is considered complete.
5. The bootstrap role structure aligns with the architecture-defined boundaries under `ansible/roles/`.

## Tasks / Subtasks

- [ ] Write failing playbook and role validation checks FIRST.
- [ ] Implement the baseline bootstrap playbook under `ansible/playbooks/`.
- [ ] Add reusable roles under `ansible/roles/` for the baseline guest setup path.
- [ ] Wire secret consumption through Vault-delivered values only.
- [ ] Verify syntax, inventory validation, and boundary compliance before completion.

## Dev Notes

- Keep provisioning concerns in OpenTofu and execution concerns in Ansible. This playbook should assume the infrastructure already exists.
- Epic 4 readiness explicitly requires Vault-delivered secret consumption only. Do not let decrypted repo material leak back into the bootstrap path.
- Bootstrap should focus on guest setup and baseline operational readiness, not on redefining infrastructure topology or provisioning resources.
- Reusable roles should align with the architecture starter layout so later operational playbooks can share patterns rather than duplicate tasks.
- Validation should cover playbook syntax, inventory compatibility, and role boundary compliance before merge.

### Project Structure Notes

- This story will primarily affect `ansible/playbooks/`, `ansible/roles/`, `tests/ansible/`, and any inventory consumption contracts established in Story 4.1.
- The architecture starter layout names `bootstrap-proxmox.yml` and `proxmox-bootstrap` as baseline targets; keep the implementation aligned with that structure.
- Secret consumption should come from Vault-delivered material and environment-scoped inventory inputs, not from repo plaintext or improvised local files.
- Keep guest/bootstrap automation reusable for later day-2 operational stories.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-42-Implement-Baseline-Guest-Bootstrap-Playbook]
- Epic 4 readiness note on Vault-delivered secret consumption: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-4.md#Findings]
- Technical decision on Ansible ownership boundary: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-004-Keep-Configuration-Management-in-Ansible]
- Architecture provisioning versus configuration boundary: [Source: docs/terminus/infra/proxmox/architecture.md#Architectural-Boundaries]
- Architecture guest/bootstrap file layout: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]
- Architecture data flow from outputs and Vault into Ansible: [Source: docs/terminus/infra/proxmox/architecture.md#Integration-Points]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 4 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Vault-only secret consumption and Ansible boundary constraints made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/4-2-implement-baseline-guest-bootstrap-playbook.md
