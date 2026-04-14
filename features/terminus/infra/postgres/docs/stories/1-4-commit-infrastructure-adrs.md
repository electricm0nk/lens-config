# Story 1.4: Commit infrastructure ADRs

Status: done

## Story

As an operator, I want the three architectural decisions recorded as ADRs, so that future maintainers understand why Patroni, pgBackRest, and the OpenTofu provider pattern were chosen.

## Acceptance Criteria

1. ADR-0002 (Patroni + Consul DCS) exists in `terminus.infra/docs/decisions/`
2. ADR-0003 (pgBackRest) exists in `terminus.infra/docs/decisions/`
3. ADR-0004 (OpenTofu cyrilgdn/postgresql + hashicorp/vault providers) exists in `terminus.infra/docs/decisions/`
4. Each ADR follows the ADR-0001 format (context, decision, consequences)

## Tasks / Subtasks

- [ ] Read ADR-0001 in `terminus.infra/docs/decisions/` to confirm format (AC: 4)
- [ ] Write ADR-0002: Patroni + Consul DCS (AC: 1, 4)
  - [ ] Context: HA requirement, Consul already in stack, alternatives considered (Etcd, repmgr)
  - [ ] Decision: Patroni with Consul DCS for automatic failover and leader election
  - [ ] Consequences: Consul dependency, no etcd, patronictl as operational tool
- [ ] Write ADR-0003: pgBackRest (AC: 2, 4)
  - [ ] Context: backup requirement, pgBackRest vs pg_dump vs WAL-G
  - [ ] Decision: pgBackRest with async WAL archiving and separate backup VM
  - [ ] Consequences: pgBackRest dependency, separate VM cost, `archive-async=y` non-blocking guarantee
- [ ] Write ADR-0004: OpenTofu cyrilgdn/postgresql + hashicorp/vault providers (AC: 3, 4)
  - [ ] Context: need database-level provisioning and Vault credential write-back
  - [ ] Decision: `cyrilgdn/postgresql` for DB/role management; `hashicorp/vault` for KV v2 write-back
  - [ ] Consequences: providers pinned in lock file; Vault connection required at apply time
- [ ] Commit all three ADRs to `terminus.infra` main (or initiative branch) (AC: 1, 2, 3)

## Dev Notes

### Architecture Context

ADRs capture decisions made in the `terminus-infra-postgres` TechPlan. ADR-0001 already exists documenting Proxmox state and Vault conventions. These three new ADRs extend that series.

- ADR location: `terminus.infra/docs/decisions/`
- Existing ADR for format reference: `docs/decisions/adr-0001-proxmox-state-and-vault-conventions.md`
- Format: Markdown with YAML front-matter (date, status, deciders) + sections: Context, Decision, Consequences

### Responsibility Split

**This story does:**
- Authors three ADR documents covering Patroni+Consul, pgBackRest, and OpenTofu providers
- Commits them to `terminus.infra/docs/decisions/`

**This story does NOT:**
- Implement any of the decisions (those are EPIC-002 and EPIC-003)
- Create runbooks (EPIC-005)

### Directory Layout

```
terminus.infra/
  docs/
    decisions/
      adr-0001-proxmox-state-and-vault-conventions.md   # EXISTING — read for format
      adr-0002-patroni-consul-dcs.md                    # CREATE
      adr-0003-pgbackrest-backup.md                     # CREATE
      adr-0004-opentofu-pg-vault-providers.md           # CREATE
```

### Testing Standards

- All three files exist in `docs/decisions/` with correct naming
- Each file has: YAML front-matter with status/date, Context section, Decision section, Consequences section
- No references to decisions not yet made (e.g., don't reference PITR decisions not committed)
- Peer-review: another operator can understand the rationale without any other context

### References

- epics.md: Story 1.4 Acceptance Criteria
- architecture.md: "3 ADRs to create: ADR-0002 (Patroni+Consul), ADR-0003 (pgBackRest), ADR-0004 (OpenTofu providers)"
- docs/terminus/infra/postgres/architecture.md: Technology Stack table for decision context
- `terminus.infra/docs/decisions/adr-0001-proxmox-state-and-vault-conventions.md` — format reference

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `docs/decisions/adr-0002-patroni-consul-dcs.md` — CREATE
- `docs/decisions/adr-0003-pgbackrest-backup.md` — CREATE
- `docs/decisions/adr-0004-opentofu-pg-vault-providers.md` — CREATE
