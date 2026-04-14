# Story 1.1: Bootstrap Repository Skeleton And Quality Baseline

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the `terminus.infra` repository initialized with the required directory structure and validation entry points,
so that all later infrastructure work starts from a verified, convention-compliant baseline.

## Acceptance Criteria

1. The repository contains the architecture-defined top-level directories: `docs/`, `tofu/`, `ansible/`, `secrets/`, `scripts/`, `tests/`, and `.github/workflows/`.
2. Placeholder README files exist for the root, `tofu/modules/`, `tofu/environments/dev/`, `tofu/environments/prod/`, `ansible/playbooks/`, and `secrets/`.
3. Validation entry points exist for `tests/tofu/`, `tests/ansible/`, and `scripts/validate.sh`.
4. A failing-first test or validation check is committed before any baseline implementation files required by this story.
5. Naming and structure in the repo baseline follow the architecture conventions exactly.

## Tasks / Subtasks

- [ ] Add failing-first validation coverage for repository structure and required placeholder files.
- [ ] Create the top-level repository directories required by the architecture and story acceptance criteria.
- [ ] Add placeholder README files for the required baseline directories.
- [ ] Add `scripts/validate.sh` plus stub validation entry points under `tests/tofu/` and `tests/ansible/`.
- [ ] Re-run the baseline validation checks and confirm the created structure matches the architecture exactly.

## Dev Notes

- Keep Story 1.1 structural only. Do not start real Consul backend setup, Vault path work, SOPS bootstrap, or any provisioning logic in this story.
- This story is the bootstrap prerequisite for all later epics. Epic 1 outputs must land before Epics 2 and 3 begin.
- Follow the architecture starter layout exactly: `tofu/modules`, `tofu/environments/{dev,prod}`, `ansible/{inventories,playbooks,roles}`, `docs`, plus the story-required `secrets`, `scripts`, `tests`, and `.github/workflows`.
- Validation expectations from architecture are lightweight at this stage: use repo-structure checks and entry points that can later run `tofu fmt`, `tofu validate`, and Ansible syntax checks.
- Preserve repo naming rules: directories in lowercase kebab-case, environment names lowercase (`dev`, `prod`), and avoid introducing implementation-specific files beyond baseline placeholders.

### Project Structure Notes

- The target repository boundary is the shared `terminus.infra` service repository, not a feature-specific repo.
- Story 1.1 should establish these baseline paths without pre-empting later stories:
  - `tofu/modules/`
  - `tofu/environments/dev/`
  - `tofu/environments/prod/`
  - `ansible/inventories/`
  - `ansible/playbooks/`
  - `ansible/roles/`
  - `secrets/`
  - `scripts/`
  - `tests/tofu/`
  - `tests/ansible/`
  - `.github/workflows/`

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-11-Bootstrap-Repository-Skeleton-And-Quality-Baseline]
- Bootstrap sequence and repository skeleton requirements: [Source: docs/terminus/infra/proxmox/epics.md#Additional-Requirements]
- Epic 1 scoping constraint: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-1.md#Findings]
- Architecture starter layout and validation guidance: [Source: docs/terminus/infra/proxmox/architecture.md#Selected-Starter-Custom-Infrastructure-Foundation-OpenTofu--Ansible]
- Repo-boundary governance: [Source: TargetProjects/lens/lens-governance/constitutions/terminus/infra/constitution.md#Article-1-Infra-Repository-Is-the-Default-Home-for-Infra-Features]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- SprintPlan generated `docs/terminus/infra/proxmox/sprint-status.yaml`.

### Completion Notes List

- Ultimate context engine analysis completed - comprehensive developer guide created.
- Story status promoted to `ready-for-dev` in sprint tracking.

### File List

- docs/terminus/infra/proxmox/sprint-status.yaml
- docs/terminus/infra/proxmox/stories/1-1-bootstrap-repository-skeleton-and-quality-baseline.md