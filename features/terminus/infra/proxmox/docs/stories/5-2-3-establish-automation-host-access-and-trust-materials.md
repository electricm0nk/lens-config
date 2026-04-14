# Story 5.2.3: Establish Automation Host Access And Trust Materials

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the automation host prepared with the required access paths and trust materials,
so that it can later run the Proxmox workflow without relying on ad hoc credentials or undocumented manual handoffs.

## Acceptance Criteria

1. The story identifies how operators securely access the automation host for day-to-day use.
2. The story documents the required trust materials and access prerequisites for the workflow, including `age` key custody expectations and Vault authentication preparation.
3. The story distinguishes between secrets or keys that stay operator-held and credentials or identities intended for automation-host use.
4. The story defines the minimum access expectations for the automation host before tooling installation or smoke execution begins.
5. The result aligns with the existing environment-scoped identity and operator-runbook guidance without introducing mixed trust boundaries.

## Tasks / Subtasks

- [ ] Document how operators reach and administer the automation host securely.
- [ ] Define which trust materials must remain operator-held versus what may be made available on the automation host.
- [ ] Capture the environment-scoped Vault auth preparation required for future secret publication and bootstrap steps.
- [ ] Record any SSH, sudo, or local administrative expectations needed for the machine to function as the control plane.
- [ ] Cross-check the story against current runbook guidance so custody and auth assumptions remain consistent.

## Dev Notes

- This story should make the control-plane trust model explicit before package installation begins.
- The existing runbook already states that `age` private keys must remain outside the repository and that Vault access must be confirmed before publication or bootstrap. This story turns those expectations into a concrete host-preparation step.
- Keep human-held trust paths and automation identities separate. Do not blur operator custody of sensitive material with environment-scoped automation identities.
- The story should stay focused on access and trust setup, not on detailed package installation or network engineering.
- If the workflow will ever use short-lived Vault auth on the automation host, document the intended preparation without hard-coding environment-specific secrets into planning artifacts.

### Project Structure Notes

- This story should primarily affect planning artifacts and operator-facing documentation under `docs/runbooks/`.
- It should build directly on the identity rules already captured in the target repo environment-identity documentation.
- Keep the resulting guidance aligned with later stories for connectivity checks, tooling installation, and smoke execution.

### References

- Story 5.2 operator-runbook context: [Source: docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md]
- Story 5.2.2 automation-host definition: [Source: docs/terminus/infra/proxmox/stories/5-2-2-create-automation-host-on-proxmox.md]
- Operator guidance on `age` custody and Vault access: [Source: TargetProjects/terminus/infra/terminus.infra/docs/runbooks/operator-runbook.md]
- Environment-scoped trust-boundary rules: [Source: TargetProjects/terminus/infra/terminus.infra/docs/runbooks/environment-identities.md]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Added to make automation-host access and trust preparation an explicit prerequisite instead of an implied operator step.

### Completion Notes List

- Separates host access and credential-custody concerns from network prep and package installation.
- Makes the control-plane trust model explicit before Story 5.2.5 and Story 5.3 execution.

### File List

- docs/terminus/infra/proxmox/stories/5-2-3-establish-automation-host-access-and-trust-materials.md