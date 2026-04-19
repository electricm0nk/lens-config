# Story 1.1: Correct portal service inventory and endpoints

Status: ready

## Story

As a platform operator,
I want the portal inventory to reflect currently reachable services and real endpoints,
so that the dashboard does not send me to dead or misleading destinations.

## Acceptance Criteria

1. The Proxmox card links to `https://10.0.0.48:8006`
2. The Proxmox health check uses `https://10.0.0.48:8006/api2/json/version`
3. Consul is removed from the service catalog
4. pgAdmin is removed from the service catalog
5. Tests or fixtures that assume the old service inventory are updated

## Tasks / Subtasks

- [ ] Update `src/config/services.js`
- [ ] Remove Consul and pgAdmin entries
- [ ] Replace Proxmox FQDN URLs with the direct IP-based URLs
- [ ] Run affected tests and update inventory assumptions where needed

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`

Primary file:
- `src/config/services.js`

Reference docs:
- `docs/terminus/portal/portal3/stories.md`
- `docs/terminus/portal/portal3/epics.md`
