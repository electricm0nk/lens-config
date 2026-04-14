# HTTPS/FQDN Migration — Terminus Portal

**Initiative:** terminus-portal-https-fqdn-fix  
**Track:** hotfix  
**Phase:** TechPlan  
**Date:** 2026-04-11

## Problem Statement

Portal dashboard tiles reference backend services using hardcoded IP addresses and mixed HTTP/HTTPS protocols, creating maintenance burden and operational risk.

## Current State (Broken)

| Service | Current URL | Protocol | Issue |
|---------|------------|----------|-------|
| Proxmox (Hypervisor) | `https://10.0.0.48:8006/` | HTTPS | IP address instead of FQDN |
| Vault (Secrets) - Primary | `https://vault.trantor.internal` | HTTPS | ✅ Correct |
| Vault (Secrets) - Legacy | `http://10.0.0.75:8200` | HTTP | IP + deprecated HTTP protocol |

## Target State (Fixed)

| Service | Target URL | Protocol | Status |
|---------|-----------|----------|--------|
| Proxmox (Hypervisor) | `https://proxmox.trantor.internal:8006/` | HTTPS | Update required |
| Vault (Secrets) | `https://vault.trantor.internal` | HTTPS | Already correct |
| Vault (Secrets) - Legacy | DELETE | — | Remove duplicate |

## Implementation Plan

### Step 1: Update Proxmox Tile
- File: `TargetProjects/terminus/portal/terminus-portal/index.html`
- Lines: 182-190
- Change: `https://10.0.0.48:8006/` → `https://proxmox.trantor.internal:8006/`

### Step 2: Remove Legacy Vault Tile
- File: `TargetProjects/terminus/portal/terminus-portal/index.html`
- Lines: 202-210
- Action: Delete duplicate/legacy Vault tile with IP address
- Rationale: Primary Vault tile (line 192) is already correct; legacy tile is now obsolete

### Step 3: Remove Legacy Fourdogs Tile
- File: `TargetProjects/terminus/portal/terminus-portal/index.html`
- Lines: 278-286
- Status: ✅ COMPLETED (deleted)

## DNS/Network Requirements

All FQDNs must resolve within `trantor.internal` domain:
- `proxmox.trantor.internal` → 10.0.0.48 (on port 8006)
- `vault.trantor.internal` → 10.0.0.75 (on port 8200)

**Assumption:** These DNS records already exist or will be created by infrastructure team (outside scope of this hotfix).

## Risk Assessment

**Low Risk** — Changes are UI-only (no application logic, no API changes).

- Portal simply navigates to these URLs in a new tab
- No credential exposure (URLs are public)
- No state changes or data modifications
- Easy rollback: revert commit

## Validation

After deployment:
1. Proxmox tile URL resolves: navigate to `https://proxmox.trantor.internal:8006/`
2. Vault tile URL resolves: navigate to `https://vault.trantor.internal`
3. Legacy Vault tile removed from dashboard
4. Legacy Fourdogs tile removed from dashboard

## Related Initiatives

- `terminus-infra-secrets` — Vault/secrets infrastructure
- `terminus-infra-proxmox` — Proxmox hypervisor
- `terminus-platform` — DNS/networking configuration

---

**Next:** Promotion to base audience for immediate deployment.
