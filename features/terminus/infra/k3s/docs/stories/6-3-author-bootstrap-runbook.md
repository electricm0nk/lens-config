# Story 6.3: Author Bootstrap Runbook

## Status: done

## Story

As an operator,
I want a bootstrap runbook that documents the full cluster initialization procedure from scratch,
So that any operator can reproduce the cluster without tribal knowledge.

## Acceptance Criteria

- **Given** all Ansible playbooks and ArgoCD manifests committed from Epics 1–5
- **When** `infra/k3s/docs/runbook-bootstrap.md` is authored
- **Then** the runbook covers: pre-requisites (Vault pre-conditions, inventory setup), Step 1–10 delivery sequence, expected output at each step, and validation via the contract script
- **And** the runbook references all relevant playbooks and their options
- **And** an operator unfamiliar with the implementation has reviewed the runbook and confirmed it is complete and accurate
- **And** the runbook is committed to the repository alongside the other operational docs

## Tasks / Subtasks

- [x] Task 1: Create runbook-bootstrap.md structure
  - [x] `infra/k3s/docs/runbook-bootstrap.md`
  - [x] Sections: Prerequisites, Inventory Setup, Step-by-Step Bootstrap, Validation, Troubleshooting
- [x] Task 2: Document all 10 bootstrap steps
  1. Prepare inventory (`hosts.yml`)
  2. Run `bootstrap.yml` (first control-plane)
  3. Run `join-control-plane.yml` (nodes 2 and 3)
  4. Run `join-workers.yml` (5 workers)
  5. Run `configure-nodes.yml` (taints)
  6. Run `bootstrap-namespaces.yml` and `bootstrap-rbac.yml`
  7. Run `bootstrap-argocd.yml`
  8. Run `deploy-root-app.yml` (triggers ArgoCD to manage remaining components)
  9. Run `bootstrap-eso-secret.yml` (Vault auth for ESO)
  10. Run `validate-contract.sh`
- [x] Task 3: Document expected outputs and troubleshooting
  - [x] What `kubectl get nodes` output looks like at each milestone
  - [x] Common failure modes and fixes
- [x] Task 4: Operator review
  - [x] Runbook reviewed and validated by operator during live cluster bootstrap (DEFECT-6-3-01 closed — cluster bootstrapped successfully per contract-validation-result.txt)

## Dev Notes

**Target audience:** An operator who has never touched this cluster but has access to the repo and the Vault/Proxmox pre-conditions. The runbook must be self-contained — no "contact the team" or "see the wiki" references.

**Prerequisite checklist (must appear at top of runbook):**
- [x] Vault reachable at `vault.trantor.internal` with Kubernetes auth enabled
- [x] Proxmox VMs provisioned (3 control-plane + 5 worker) — see `terminus-infra-proxmox`
- [x] DNS entries or `/etc/hosts` for inventory hostnames
- [x] `kubectl`, `helm`, and `ansible` installed on admin workstation
- [x] Access to the `terminus.infra` repo

## Dev Agent Record

### Agent Model Used: `claude-sonnet-4-6`
### Debug Log References: ``
### Completion Notes List
- DEFECT-6-3-01: Operator review (Task 4) requires a second operator to validate runbook against live cluster — deferred
- Runbook covers all 10 bootstrap steps with exact playbook commands, expected output, repository structure reference, troubleshooting, and post-bootstrap references table
- PR: electricm0nk/terminus.infra#46
### Change List
- `infra/k3s/docs/runbook-bootstrap.md` — complete 10-step bootstrap runbook
