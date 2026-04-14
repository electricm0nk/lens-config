# Story 5.2.2: Create Automation Host On Proxmox

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want a dedicated automation host VM on Proxmox,
so that the infrastructure workflow has a stable control-plane machine for OpenTofu, Vault, SOPS, inventory rendering, and Ansible execution.

## Acceptance Criteria

1. The story defines the automation host as a control-plane VM separate from the Proxmox hypervisor itself.
2. Baseline VM expectations are documented, including Ubuntu version, sizing, storage, and network placement assumptions.
3. The story identifies the control-plane responsibilities the host must support for the current repo workflow.
4. The story notes the trust-boundary and failure-domain tradeoffs of hosting the automation machine inside the same Proxmox environment it manages.
5. The result is sufficient to support the later tooling-installation and smoke-validation stories without hidden infrastructure assumptions.

## Tasks / Subtasks

- [ ] Define the intended role of the automation host in the Proxmox workflow.
- [ ] Document baseline VM assumptions for CPU, memory, disk, Ubuntu release, and network placement.
- [ ] Map the host responsibilities to the repo command surface: validation, publication, inventory rendering, and Ansible execution.
- [ ] Capture trust-boundary and recovery tradeoffs for running the controller inside Proxmox.
- [ ] Record any dependencies on prior network or access stories that must be satisfied before host creation.

## Dev Notes

- This story is about creating the control-plane VM concept and requirements, not yet the full package/tool bootstrap inside it.
- The automation host should be treated as an admin/control machine, not as part of the managed application workload.
- A Proxmox-resident automation host improves repeatability, but it also creates a circular dependency if the platform is unhealthy; that tradeoff should be stated explicitly.
- Keep the host separate from the hypervisor node itself. Installing the workflow toolchain directly on the Proxmox host should be treated as an exception path, not the default design.
- Later stories can attach package installation and actual smoke execution to this host once connectivity and credentials are in place.

### Project Structure Notes

- This story should primarily affect planning artifacts and operator-facing documentation under `docs/runbooks/` if implemented.
- It should sit between hypervisor-network readiness and control-host tooling installation in the execution sequence.
- Keep references aligned with the existing operator runbook and readiness harness rather than inventing a separate workflow.

### References

- Story 5.2 operator-runbook context: [Source: docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md]
- Story 5.2.5 Ubuntu control-host tooling dependency: [Source: docs/terminus/infra/proxmox/stories/5-2-5-document-ubuntu-control-host-tooling-packages.md]
- Story 5.3 live smoke dependency on a real control environment: [Source: docs/terminus/infra/proxmox/stories/5-3-execute-dev-smoke-validation-and-capture-readiness-evidence.md]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Added after deciding that a dedicated Proxmox-resident automation host is the preferred self-contained control-plane design.

### Completion Notes List

- Makes the automation host an explicit planning artifact instead of an implicit assumption.
- Separates VM creation concerns from later tooling installation and live smoke validation.

### File List

- docs/terminus/infra/proxmox/stories/5-2-2-create-automation-host-on-proxmox.md