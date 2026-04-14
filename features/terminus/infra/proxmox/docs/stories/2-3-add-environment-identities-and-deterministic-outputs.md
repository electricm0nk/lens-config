# Story 2.3: Add Environment Identities And Deterministic Outputs

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want per-environment automation identities and deterministic OpenTofu outputs,
so that downstream Ansible workflows can consume infrastructure metadata without reading raw state.

## Acceptance Criteria

1. Dev and prod identity inputs are defined separately and documented as distinct trust boundaries.
2. Output values needed by downstream automation are exposed from the environment roots with stable names.
3. No script or playbook in the story parses raw tfstate to obtain provisioning data.
4. Output contracts cover at least host addressing, inventory grouping inputs, and secret publication context required downstream.
5. Tests or contract checks exist for output names and required fields before final implementation is completed.

## Tasks / Subtasks

- [ ] Write failing contract tests FIRST for required outputs, output names, and environment identity separation.
- [ ] Add environment-scoped identity inputs and documentation that preserve the `dev` versus `prod` trust boundary.
- [ ] Expose stable OpenTofu outputs for host addressing, inventory grouping, and secret publication context.
- [ ] Ensure scripts and playbooks consume explicit outputs rather than parsing raw state.
- [ ] Verify output names remain deterministic across environments and narrow enough for downstream automation.

## Dev Notes

- Story 2.3 is the handoff contract between provisioning and downstream automation. Keep the contract narrow, explicit, and environment-scoped.
- Epic 2 readiness warns that ad hoc outputs will make Story 4 brittle. Expose only the outputs later inventory rendering and bootstrap actually need.
- Environment identities are part of the trust boundary model, not just variable naming. Keep `dev` and `prod` distinct in inputs, outputs, and documentation.
- Never solve downstream integration by permitting raw tfstate parsing. The architecture explicitly rejects that pattern.
- Output names should be descriptive `snake_case` and stable enough for scripts and playbooks to consume without interpretation.

### Project Structure Notes

- This story should primarily affect `tofu/environments/dev/`, `tofu/environments/prod/`, module outputs under `tofu/modules/`, and contract validation under `tests/tofu/`.
- The downstream consumers of these outputs are inventory rendering, bootstrap inputs, and secret publication context.
- Keep output generation inside OpenTofu; scripts may transform outputs, but they should not become alternate metadata authorities.
- Preserve environment isolation across identity, state, output, and inventory boundaries.

### References

- Story definition and acceptance criteria: [Source: docs/terminus/infra/proxmox/epics.md#Story-23-Add-Environment-Identities-And-Deterministic-Outputs]
- Epic 2 readiness note on narrow output contracts: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-2.md#Findings]
- Technical decision on per-environment machine identities: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-008-Use-Per-Environment-Machine-Identities]
- Technical decision on output contracts: [Source: docs/terminus/infra/proxmox/tech-decisions.md#TD-010-OpenTofu-Outputs-Are-the-Integration-Handoff-Contract]
- Architecture naming and handoff rules for outputs: [Source: docs/terminus/infra/proxmox/architecture.md#File-Organization-Patterns]
- Architecture integration boundary rejecting raw state scraping: [Source: docs/terminus/infra/proxmox/architecture.md#Integration-Points]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Story enriched from planning artifacts, architecture, and Epic 2 readiness guidance.

### Completion Notes List

- Story upgraded to the richer developer handoff format.
- Output-contract and identity-boundary constraints made explicit for downstream stories.

### File List

- docs/terminus/infra/proxmox/stories/2-3-add-environment-identities-and-deterministic-outputs.md
