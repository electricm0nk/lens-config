# Story 5.2.5: Document Ubuntu Control Host Tooling Packages

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As an operator,
I want a concrete Ubuntu control-host package list for the Proxmox automation workflow,
so that I can prepare one machine with the client-side tooling needed to run provisioning, secret publication, inventory rendering, and bootstrap commands.

## Acceptance Criteria

1. The story clearly states that the required toolchain is installed on a control machine, not on the Proxmox node by default.
2. The package list identifies the required Ubuntu-side tools for OpenTofu, Vault, SOPS, Ansible, and the surrounding helper utilities.
3. The story distinguishes between packages normally available from Ubuntu repositories and tools commonly installed from vendor repositories or upstream package sources.
4. The documented package set is sufficient to support `./scripts/validate.sh`, `./scripts/publish-vault-secrets.sh`, `./scripts/render-inventory.sh`, and the Ansible playbook commands used in the runbook.
5. The story includes operator notes about network access and credentials so the package list is not mistaken for the entire readiness requirement.

## Tasks / Subtasks

- [ ] Document the control-host role explicitly so operators do not confuse the managed Proxmox server with the machine that runs the toolchain.
- [ ] List the Ubuntu packages or package-source expectations for the required client-side tools.
- [ ] Separate baseline distro packages from tools that typically come from vendor repositories or direct package installs.
- [ ] Cross-reference the validation, publication, inventory, and bootstrap commands that depend on the toolchain.
- [ ] Add the resulting package guidance to the operator-facing runbook or an adjacent runbook section.

## Dev Notes

- This story is documentation-only. It does not require implementation changes in the target repo automation paths.
- The key clarification is control machine versus managed Proxmox host. The toolchain belongs on the control side unless the Proxmox machine is intentionally being used as that control node.
- The minimum practical Ubuntu control-host toolchain for the current workflow is:
  - `git`
  - `curl`
  - `ca-certificates`
  - `gnupg`
  - `jq`
  - `python3`
  - `python3-pip`
  - `age`
  - `ansible-core`
  - `vault`
  - `sops`
  - `opentofu`
- `vault`, `sops`, and `opentofu` may require vendor-managed APT repositories or upstream package sources rather than stock Ubuntu repositories, depending on the target Ubuntu release and current package availability.
- The package list alone is not enough for Story 5.3. The control host will also need network reachability to Proxmox and Vault plus the required operator-managed credentials.

### Project Structure Notes

- This story should primarily affect operator-facing documentation under `docs/runbooks/`.
- Keep the package list tied to the existing command surface:
  - `./scripts/validate.sh`
  - `./scripts/publish-vault-secrets.sh <environment>`
  - `./scripts/render-inventory.sh <environment>`
  - `ansible-playbook -i ansible/inventories/<environment>/hosts.yml ...`
- Avoid turning this into a full machine-bootstrap script unless a later story explicitly calls for that.

### References

- Story 5.2 operator-runbook context: [Source: docs/terminus/infra/proxmox/stories/5-2-document-operator-runbook-and-recovery-paths.md]
- Story 5.3 smoke-validation dependency on real toolchain access: [Source: docs/terminus/infra/proxmox/stories/5-3-execute-dev-smoke-validation-and-capture-readiness-evidence.md]
- Epic 5 readiness note on operator-managed steps: [Source: docs/terminus/infra/proxmox/implementation-readiness-epic-5.md#Findings]
- Operator workflow commands already documented in the target repo runbooks: [Source: TargetProjects/terminus/infra/terminus.infra/docs/runbooks/operator-runbook.md]

## Dev Agent Record

### Agent Model Used

GPT-5.4

### Debug Log References

- Renumbered from Story 5.2.1 after splitting out hypervisor network and automation-host planning concerns.

### Completion Notes List

- Captures the Ubuntu-side package expectations for a control host.
- Clarifies that Story 5.3 depends on both tooling and real environment access.

### File List

- docs/terminus/infra/proxmox/stories/5-2-5-document-ubuntu-control-host-tooling-packages.md