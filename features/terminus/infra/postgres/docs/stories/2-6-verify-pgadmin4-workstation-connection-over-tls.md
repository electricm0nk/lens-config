# Story 2.6: Verify pgAdmin4 workstation connection over TLS

Status: ready-for-dev

## Story

As an operator, I want to open pgAdmin4 on my workstation and connect to `master.postgresql.service.consul:5432` over TLS, so that FR7 is verified and I have a working management interface.

## Acceptance Criteria

1. pgAdmin4 connects using `sslmode=require`
2. Connection uses break-glass admin credential retrieved from Vault (`secret/terminus/{env}/postgres/admin`)
3. pgAdmin4 shows database list and reports server version 17.x
4. Plaintext connection attempt (`sslmode=disable`) from pgAdmin4 is rejected

## Tasks / Subtasks

- [ ] Retrieve break-glass admin credential from Vault (AC: 2)
  - [ ] `vault kv get secret/terminus/{env}/postgres/admin` — extract username and password
  - [ ] Confirm credential exists (created during Patroni bootstrap in Story 2.2)
- [ ] Add pgAdmin4 server connection in pgAdmin4 on workstation (AC: 1, 2, 3)
  - [ ] Host: `master.postgresql.service.consul` (or primary VM IP if DNS not reachable from workstation)
  - [ ] Port: `5432`
  - [ ] SSL mode: `require`
  - [ ] Username and password from Vault admin credential
- [ ] Verify connection and server info (AC: 3)
  - [ ] pgAdmin4 "Dashboard" shows server version: `PostgreSQL 17.x`
  - [ ] Database list shows at least `postgres` database
- [ ] Attempt plaintext connection from pgAdmin4 (AC: 4)
  - [ ] Set pgAdmin4 SSL mode to `disable`
  - [ ] Confirm connection fails with SSL error
- [ ] Document outcome in story completion notes

## Dev Notes

### Architecture Context

pgAdmin4 runs on the operator's workstation. `master.postgresql.service.consul` provides dynamic routing to the current Patroni primary. If the workstation's DNS does not resolve Consul service discovery, use the primary VM's static IP as a fallback.

- Vault path: `secret/terminus/{env}/postgres/admin`
- pgAdmin4 SSL mode setting: connection properties → SSL tab → SSL mode = `require`
- This is a verification story with no code changes to `terminus.infra`
- Break-glass credential: admin-level superuser credential in Vault for operator access only

### Responsibility Split

**This story does:**
- Retrieves break-glass credential from Vault
- Connects pgAdmin4 over TLS and verifies server version
- Attempts and confirms plaintext rejection

**This story does NOT:**
- Install pgAdmin4 (assumed pre-installed on operator workstation)
- Automate pgAdmin4 configuration (manual workstation config is acceptable)
- Create application-level database users (EPIC-004)

### Directory Layout

No `terminus.infra` files modified. Outcome recorded in story completion notes only.

If Consul DNS is not reachable from workstation, document workaround:
- Use `ssh -L 5432:postgres-primary-vm:5432 automation-server` tunnel

### Testing Standards

- pgAdmin4 connection tab shows "Connected" with no error
- Server info shows PostgreSQL 17.x version string
- Database browser shows system databases
- `sslmode=disable` attempt shows connection error in pgAdmin4

### References

- epics.md: Story 2.6 Acceptance Criteria
- architecture.md: FR7 — Operator can access pgAdmin4 on workstation and connect over TLS
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/admin (break-glass)"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

(no terminus.infra files — workstation verification only)
