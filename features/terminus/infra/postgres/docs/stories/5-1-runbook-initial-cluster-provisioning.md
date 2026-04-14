# Story 5.1: Runbook: Initial cluster provisioning

Status: ready-for-dev

## Story

As an operator, I want a runbook covering end-to-end initial provisioning from zero, so that the cluster can be stood up by following steps alone without prior context.

## Acceptance Criteria

1. Runbook covers: prerequisites check → SOPS key setup → `tofu apply` → bootstrap verification → Patroni health check
2. Every command is copy-pasteable with no placeholders requiring prior context
3. Stored at `docs/terminus/infra/postgres/runbooks/01-initial-provisioning.md`

## Tasks / Subtasks

- [ ] Outline runbook sections (AC: 1)
  - [ ] Section 1: Prerequisites — Verify OpenTofu, Vault CLI, SOPS, age key on automation server
  - [ ] Section 2: SOPS key setup — Confirm age key exists; `sops -d terraform.tfstate.enc > terraform.tfstate`
  - [ ] Section 3: `tofu apply` — Navigate to `tofu/environments/dev/`; run `tofu plan` then `tofu apply`
  - [ ] Section 4: Bootstrap verification — SSH to VMs, verify hostnames, SSH from each node
  - [ ] Section 5: Patroni health check — `patronictl list`, confirm Leader + Replica Running
  - [ ] Section 6: Post-provisioning checks — Vault credentials exist, Consul DNS resolves
- [ ] Write each runbook section with copy-pasteable commands (AC: 2)
  - [ ] All `{env}` substitutions replaced with literal `dev` (or clearly noted as the only variable)
  - [ ] All hostnames and paths as they appear in the actual deployment
- [ ] Commit runbook to `terminus.infra` (AC: 3)

## Dev Notes

### Architecture Context

This runbook documents the procedure executed across Stories 1.1–2.2. It is the canonical starting point for a fresh cluster deployment and should be executable without reading any epics, PRDs, or architecture documents.

- Target repo: `terminus.infra`
- Runbook path: `docs/terminus/infra/postgres/runbooks/01-initial-provisioning.md` (note: create directory if needed)
- Commands reference: `tofu apply`, SOPS decrypt, `patronictl`, `vault kv get`

### Responsibility Split

**This story does:**
- Authors the initial provisioning runbook

**This story does NOT:**
- Author other runbooks (5.2–5.8)
- Implement any provisioning code (EPIC-001, EPIC-002)

### Directory Layout

```
terminus.infra/
  docs/
    terminus/
      infra/
        postgres/
          runbooks/
            01-initial-provisioning.md    # CREATE
```

### Testing Standards

- Runbook reviewed by a second person (or author themselves after 24h) and all commands verified as executable
- No placeholders of the form `<REPLACE_ME>` in commands — all values are literal or clearly identified as the only environment variable (`{env}` → `dev`)
- Peer review: can another operator follow this runbook without asking the author any questions?

### References

- epics.md: Story 5.1 Acceptance Criteria
- architecture.md: FR21 — All operations documented as step-by-step runbooks executable without prior context
- architecture.md: NFR7 — Full cluster reprovisioning from git with no manual steps beyond age key

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/terminus/infra/postgres/runbooks/01-initial-provisioning.md` — CREATE
