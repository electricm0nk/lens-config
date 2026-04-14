# Story 5.2.4: Validate Automation Host Connectivity And Readiness

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the automation host connectivity verified before toolchain setup and smoke execution,
so that workflow failures are caught as environment-readiness issues instead of surfacing later as misleading provisioning or bootstrap errors.

## Acceptance Criteria

1. The story defines the required outbound and management connectivity checks for the automation host.
2. The verification covers the minimum destinations needed by the repo workflow, including Proxmox, Vault, backend services, and package sources.
3. The story documents how readiness failures should be classified so they map cleanly to the existing readiness and recovery guidance.
4. The story clarifies that connectivity validation happens before relying on tool installation or live smoke evidence collection.
5. The result is sufficient to support later package-installation and smoke-validation work without hidden network or reachability assumptions.

## Tasks / Subtasks

- [ ] Enumerate the minimum destinations the automation host must reach for the repo workflow.
- [ ] Define the expected connectivity checks for management APIs, secrets systems, state backends, and package sources.
- [ ] Document how failures should be classified so operators can distinguish network-readiness issues from tooling or workflow issues.
- [ ] Tie the readiness outcome back to the standard runbook sequence and later smoke-validation work.
- [ ] Record any unresolved dependencies that block the automation host from acting as a usable control plane.

## Dev Notes

- This story is the bridge between infrastructure preparation and actual tool use.
- The target repo already classifies failures by readiness stage and recovery area. Keep this story aligned with those stage boundaries rather than inventing a new troubleshooting taxonomy.
- The minimum destinations include the Proxmox API, Vault, remote state/backend services, and package sources required to install and maintain the control-plane toolchain.
- Treat missing reachability as a first-class readiness failure. Do not defer it until `tofu`, secret publication, or Ansible commands start failing later.
- Keep this story scoped to validating access paths and readiness, not to scripting the full smoke workflow.

### Project Structure Notes

- This story should primarily affect planning artifacts and operator-facing documentation under `docs/runbooks/`.
- It should align with the target repo readiness stages and recovery guidance already documented for repeated operation.
- Keep the outputs discoverable so Story 5.3 can reference them as part of a stable evidence chain if needed.

### References

- Story 5.2 operator-runbook context: [Source: docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md]
- Story 5.2.1 hypervisor network prerequisite: [Source: docs/terminus/infra/proxmox/stories/5-2-1-evaluate-and-update-hypervisor-network-settings.md]
- Story 5.2.3 access and trust prerequisite: [Source: docs/terminus/infra/proxmox/stories/5-2-3-establish-automation-host-access-and-trust-materials.md]
- Readiness stage guidance: [Source: TargetProjects/terminus/infra/terminus.infra/docs/runbooks/readiness-validation.md]
- Operator workflow destinations and recovery points: [Source: TargetProjects/terminus/infra/terminus.infra/docs/runbooks/operator-runbook.md]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Added to make automation-host reachability verification a discrete planning step before tooling and smoke execution.

### Completion Notes List

- Separates connectivity validation from both trust setup and package installation.
- Aligns the future automation-host workflow with the repo’s existing readiness-stage model.

### File List

- docs/terminus/infra/proxmox/stories/5-2-4-validate-automation-host-connectivity-and-readiness.md