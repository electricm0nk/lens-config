# Story 5.2.1: Evaluate And Update Hypervisor Network Settings

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want the Proxmox hypervisor networking reviewed and adjusted for automation-host support,
so that the future control VM can reach Proxmox, Vault, and other required services on stable and intended network paths.

## Acceptance Criteria

1. The story documents the current hypervisor-side network shape relevant to an automation host VM, including the intended management bridge, addressing assumptions, and network attachment point.
2. Required management, API, and outbound connectivity paths are documented for the control-plane workflow, including Proxmox API access, Vault access, backend/state access, and package-repository reachability.
3. Any needed bridge, VLAN, gateway, DNS, firewall, or routing adjustments are described at the hypervisor level before automation-host creation begins.
4. The story distinguishes between changes required on the Proxmox hypervisor and changes that belong inside the future automation host guest.
5. The story includes a safe execution sequence, rollback expectation, and access-protection notes so hypervisor network changes do not strand operator access.
6. The result produces a stable network profile that later stories can reference for automation-host provisioning, host-readiness validation, and smoke execution.

## Tasks / Subtasks

- [ ] Review the current Proxmox hypervisor networking relevant to a future automation VM, including bridges, VLAN usage, addressing assumptions, and default egress path.
- [ ] Identify the minimum connectivity needed for OpenTofu, Vault, SOPS workflows, Ansible, Consul backend access, and package installation.
- [ ] Document any required bridge, VLAN, gateway, DNS, firewall, or routing changes before automation-host provisioning.
- [ ] Define the intended network attachment for the automation host VM so Story 5.2.2 can create the guest on a known-good segment.
- [ ] Separate hypervisor network work from guest OS configuration work.
- [ ] Record the safe change sequence, rollback approach, and any console or out-of-band access required before making network changes.
- [ ] Record any unresolved network dependencies that would block the automation host from operating as the control plane.

## Dev Notes

- This story exists because the control-host toolchain only matters if the machine can actually reach the required management endpoints.
- The required connectivity includes, at minimum, the Proxmox API, Vault, package repositories, and any remote backend services used by the workflow.
- The current architecture and technical decisions intentionally defer exact network segmentation and SDN layout. This story is where those deferred choices should be resolved enough to support execution.
- Avoid collapsing hypervisor network evaluation into guest provisioning. The goal here is to make the underlying VM placement and connectivity viable first.
- Protect existing operator access while making changes. If network edits could disrupt current Proxmox management access, the plan should require a safe rollback path and console access before changes are attempted.
- If the intended automation host will live on an isolated management network, document how package installation and Vault access are expected to work from that segment.
- Keep this operator-facing and infrastructure-scoped; it should not drift into application-network planning.
- The ideal output of this story is a stable network profile that later stories can reference directly, not a one-time troubleshooting note.

### Project Structure Notes

- This story should primarily affect planning artifacts and operator-facing documentation under `docs/runbooks/` if implemented.
- It is a prerequisite-style story for the automation-host path and for the later smoke-validation work.
- Keep the outputs of this story stable and discoverable so later stories can assume the network shape rather than rediscover it.
- The resulting guidance should be usable by Story 5.2.2 for host placement, Story 5.2.4 for reachability checks, and Story 5.3 for live smoke execution context.

### References

- Story 5.2 operator-runbook context: [Source: docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md]
- Story 5.2.2 automation-host dependency on a known network attachment: [Source: docs/terminus/infra/proxmox/stories/5-2-2-create-automation-host-on-proxmox.md]
- Technical decision log deferred network-segmentation choice: [Source: docs/terminus/infra/proxmox/tech-decisions.md#Deferred-Decisions]
- Story 5.3 live smoke dependency on real network reachability: [Source: docs/terminus/infra/proxmox/stories/5-3-execute-dev-smoke-validation-and-capture-readiness-evidence.md]
- Epic 5 readiness note on operator-managed requirements: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-5.md#Findings]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Added after clarifying that the project may need hypervisor-side network work before an automation host can be useful.

### Completion Notes List

- Separates hypervisor network evaluation from guest automation-host setup.
- Makes network reachability an explicit prerequisite for the self-contained control-plane path.
- Makes rollback and access-protection part of the handoff so network changes can be executed safely.

### File List

- docs/terminus/infra/proxmox/stories/5-2-1-evaluate-and-update-hypervisor-network-settings.md