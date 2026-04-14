# Story 2.2: Bootstrap Patroni cluster with Consul DCS

Status: ready-for-dev

## Story

As an operator, I want Patroni initialized with Consul as the DCS and both nodes joined to the cluster, so that I have a functioning primary + replica pair with automatic leader election.

## Acceptance Criteria

1. `patronictl -c /etc/patroni/patroni.yml list` shows one Leader and one Replica, both healthy
2. Patroni scope is `postgres-{env}`
3. Consul service `master.postgresql.service.consul` resolves to primary
4. `patroni-consul-token` written to `secret/terminus/{env}/postgres/patroni-consul-token` in Vault

## Tasks / Subtasks

- [ ] Write `patroni.yml` template as `remote-exec` delivered file (AC: 1, 2)
  - [ ] Set `scope: postgres-{env}` using environment variable
  - [ ] Configure `[consul]` DCS section: address, scheme, token reference
  - [ ] Configure `[postgresql]` section: data_dir, bin_dir, connect_address, ports
  - [ ] Configure `[bootstrap]` section: pg_hba rules (hostssl), pg_ctl_timeout
- [ ] Generate Consul ACL token for Patroni and write to Vault (AC: 4)
  - [ ] Create Consul ACL token with `session:write` and `service:write` permissions
  - [ ] Write token to `secret/terminus/{env}/postgres/patroni-consul-token` via `hashicorp/vault` provider
- [ ] Add `remote-exec` provisioner to deliver and apply `patroni.yml` on both nodes (AC: 1, 2)
  - [ ] Deliver `patroni.yml` to `/etc/patroni/patroni.yml`
  - [ ] Set ownership: `postgres:postgres`
  - [ ] Enable and start `patroni.service`: `systemctl enable patroni && systemctl start patroni`
- [ ] Wait for cluster initialization and verify leader election (AC: 1)
  - [ ] After primary node provisioned, allow replica `remote-exec` to run second
  - [ ] Add `depends_on` ordering: replica `remote-exec` depends on primary Patroni start
- [ ] Verify Consul DNS resolves after cluster forms (AC: 3)
  - [ ] `dig @consul-server master.postgresql.service.consul` returns primary IP

## Dev Notes

### Architecture Context

Consul is already deployed in Terminus (version 1.22.5). Patroni will use Consul as its DCS for distributed leader election. The Consul service tag `master` automatically routes to the primary node once Patroni registers.

- Patroni scope: `postgres-{env}` — e.g., `postgres-dev`
- Consul DCS endpoint: existing Consul cluster address
- Vault path for token: `secret/terminus/{env}/postgres/patroni-consul-token`
- PostgreSQL data dir: `/var/lib/postgresql/17/main`
- `patroni.service` systemd unit required (may need to create if not bundled with package)

### Responsibility Split

**This story does:**
- Writes `patroni.yml` with Consul DCS config on both cluster nodes
- Creates Consul ACL token and writes to Vault
- Starts Patroni on both nodes and verifies cluster formation

**This story does NOT:**
- Install packages (Story 2.1)
- Configure TLS in `pg_hba.conf` (Story 2.3 — though `patroni.yml` bootstrap section may include placeholder hostssl rules)
- Configure pgaudit (Story 2.4)

### Directory Layout

```
terminus.infra/
  tofu/
    modules/
      postgres-cluster/
        main.tf              # MODIFY — add patroni.yml delivery + systemd provisioners
        templates/
          patroni.yml.tpl    # CREATE — Patroni config template
        scripts/
          start-patroni.sh   # CREATE — systemctl enable/start patroni
```

### Testing Standards

- `patronictl -c /etc/patroni/patroni.yml list` on primary shows: Leader (Running), Replica (Running)
- Patroni scope in output matches `postgres-{env}`
- `dig @127.0.0.1 master.postgresql.service.consul` resolves to primary VM IP
- `vault kv get secret/terminus/dev/postgres/patroni-consul-token` returns a token value
- `tofu apply` re-run produces no changes

### References

- epics.md: Story 2.2 Acceptance Criteria
- architecture.md: "HA tooling: Patroni + Consul DCS; Consul 1.22.5 already in stack"
- architecture.md: "Naming conventions: Patroni scope postgres-{env}; Consul service master.postgresql.service.consul"
- architecture.md: "Vault paths: secret/terminus/{env}/postgres/patroni-consul-token"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List

- `tofu/modules/postgres-cluster/main.tf` — MODIFY (patroni.yml delivery + systemd start provisioners)
- `tofu/modules/postgres-cluster/templates/patroni.yml.tpl` — CREATE
- `tofu/modules/postgres-cluster/scripts/start-patroni.sh` — CREATE
