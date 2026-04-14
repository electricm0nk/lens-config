# Story 2.3: Enforce TLS-only connections

Status: ready-for-dev

## Story

As an operator, I want PostgreSQL to accept only TLS connections and reject plaintext, so that NFR4 is satisfied and all traffic is encrypted at the listener.

## Acceptance Criteria

1. `ssl = on` in `postgresql.conf`
2. `pg_hba.conf` contains only `hostssl` entries; no plaintext `host` entries exist
3. `psql --no-ssl` connection attempt is rejected by the server
4. Server cert is sourced from infra TLS CA

## Tasks / Subtasks

- [ ] Retrieve or generate server TLS cert from infra TLS CA (AC: 4)
  - [ ] Cert and key stored in Vault or delivered via `remote-exec` provisioner
  - [ ] Place cert at `$PGDATA/server.crt` and key at `$PGDATA/server.key`
  - [ ] Set ownership and permissions: `postgres:postgres`, key mode `0600`
- [ ] Configure `ssl = on` in `postgresql.conf` (AC: 1)
  - [ ] `ssl_cert_file` and `ssl_key_file` pointing to cert/key paths
  - [ ] Delivered via `remote-exec` provisioner that appends/replaces settings
- [ ] Update Patroni `patroni.yml` bootstrap `pg_hba` section to `hostssl` only (AC: 2)
  - [ ] Replace any `host` entries with `hostssl` in `patroni.yml.tpl`
  - [ ] Ensure replication entry also uses `hostssl`
- [ ] Reload PostgreSQL config after changes (AC: 1, 2)
  - [ ] `patronictl reload postgres-{env}` or `SELECT pg_reload_conf()`
- [ ] Verify TLS rejection of plaintext connections (AC: 3)
  - [ ] Attempt `psql "host=postgres-primary-vm sslmode=disable"` from automation server
  - [ ] Confirm connection is rejected with `SSL off` error

## Dev Notes

### Architecture Context

TLS enforcement is a hard security requirement (NFR4, FR8). The infra TLS CA is the existing certificate authority in the Terminus stack. PostgreSQL TLS is configured at the `postgresql.conf` level with `pg_hba.conf` enforcing `hostssl` for all connections.

- TLS cert source: infra TLS CA (Terminus existing PKI)
- Cert placement: `$PGDATA/server.crt` and `$PGDATA/server.key`
- `$PGDATA`: `/var/lib/postgresql/17/main`
- Patroni manages `postgresql.conf` and `pg_hba.conf` — changes must go through `patroni.yml` or `patronictl edit-config`

### Responsibility Split

**This story does:**
- Configures `ssl = on` and cert/key paths in `postgresql.conf`
- Enforces `hostssl`-only entries in `pg_hba.conf`
- Sources cert from infra TLS CA

**This story does NOT:**
- Set up the infra TLS CA (existing infrastructure)
- Configure client TLS certificates (password auth with TLS transport is sufficient)
- Configure pgadmin4 TLS client settings (Story 2.6)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf              # MODIFY — add TLS cert delivery provisioner
        templates/
          patroni.yml.tpl    # MODIFY — update pg_hba to hostssl entries
        scripts/
          configure-tls.sh   # CREATE — place certs, set permissions, reload PG
```

### Testing Standards

- `SHOW ssl;` in psql returns `on`
- `psql "sslmode=disable" -h postgres-primary-vm` returns: `psql: error: ... SSL off ...`
- `psql "sslmode=require" -h postgres-primary-vm` connects successfully
- `cat $PGDATA/pg_hba.conf | grep -v hostssl | grep '^host'` returns empty (no plaintext host lines)
- Replica connection to primary also uses TLS (streaming replication hostssl)

### References

- epics.md: Story 2.3 Acceptance Criteria
- architecture.md: "TLS: Server cert from infra TLS CA; ssl=on; pg_hba.conf hostssl-only entries"
- architecture.md: NFR4 — All PostgreSQL connections must use TLS; plaintext connections rejected at the listener
- architecture.md: FR8 — PostgreSQL accepts TLS connections only

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (TLS cert delivery provisioner)
- `tofu/modules/postgres-cluster/templates/patroni.yml.tpl` — MODIFY (hostssl pg_hba entries)
- `tofu/modules/postgres-cluster/scripts/configure-tls.sh` — CREATE
