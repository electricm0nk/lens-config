# Story 4.1: Render Inventory From OpenTofu Outputs

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want generated inventory or structured bootstrap inputs derived from OpenTofu outputs,
so that Ansible can target the provisioned infrastructure without scraping raw state.

## Acceptance Criteria

1. Inventory generation consumes only documented OpenTofu outputs from Story 2.3.
2. The generated inventory path and naming are deterministic and environment-scoped.
3. Inventory structure is valid for the baseline bootstrap playbooks.
4. Inventory generation fails clearly if required outputs are missing.
5. Tests or validation checks for inventory rendering are written before the final implementation for this story.

## Tasks / Subtasks

- [ ] Write failing inventory-render tests FIRST for required outputs and deterministic file layout.
- [ ] Implement inventory rendering from OpenTofu outputs only.
- [ ] Validate generated inventory against the intended bootstrap playbook input shape.
- [ ] Add clear failure messages for missing or malformed outputs.
- [ ] Verify inventory output is environment-scoped, reproducible, and suitable for downstream bootstrap playbooks.

## Dev Notes

- This is the contract bridge between provisioning and Ansible. Do not parse local state files here under any circumstance.
- Epic 4 readiness treats hard failure on missing outputs as essential. Silent fallback behavior would effectively reintroduce raw-state scraping through side channels.
- Use the deterministic output contract from Story 2.3 exactly; do not infer fields or backfill missing data with ad hoc defaults.
- Generated inventory should remain machine-readable, environment-scoped, and stable enough for repeated bootstrap execution.
- Keep this story focused on rendering and validation, not on guest bootstrap logic itself.

### Project Structure Notes

- This story will likely affect `scripts/render-inventory.sh`, `tests/ansible/inventory.sh`, generated inventory locations under `ansible/inventories/`, and any supporting output-consumption helpers.
- Inventory input must come from explicit OpenTofu outputs, not raw state or hand-maintained operator files.
- Keep generated group names lowercase and stable in line with the architecture’s naming guidance.
- The resulting inventory must align with the bootstrap playbook shape expected in Story 4.2.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-41-Render-Inventory-From-OpenTofu-Outputs]
- Epic 4 readiness note on failing hard for missing outputs: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-4.md#Findings]
- Technical decision on output contracts: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-010-OpenTofu-Outputs-Are-the-Integration-Handoff-Contract]
- Architecture integration rule for output-driven inventory: [Source: docs/terminus/infra/proxmox/architecture.md#Integration-Points]
- Architecture naming and output conventions: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]
- Architecture example file placement for inventory rendering: [Source: docs/terminus/infra/proxmox/architecture.md#Complete-Project-Directory-Structure]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 4 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Missing-output failure behavior and output-only inventory sourcing made explicit for implementation.

### File List

- docs/terminus/infra/proxmox/stories/4-1-render-inventory-from-opentofu-outputs.md
