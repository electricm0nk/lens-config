---
feature: portal3
doc_type: implementation-readiness
status: draft
goal: "Assess whether portal3 is sufficiently bounded, understood, and verified to hand off into implementation."
key_decisions:
  - "Implementation is ready to proceed as a focused UI and configuration iteration in the existing portal repo."
open_questions:
  - "Whether private GitHub visibility is needed immediately or can wait for a backend-assisted follow-up."
depends_on: []
blocks: []
updated_at: 2026-04-19T02:18:00Z
---

# Implementation Readiness — portal3

## Summary

| Category | Status |
|---|---|
| Scope | ✅ READY |
| Architecture | ✅ READY |
| Repo Context | ✅ READY |
| Verification Path | ✅ READY |
| Operational Risks | ⚠️ NOTED |
| **Overall** | **✅ PROCEED** |

## Readiness Assessment

### Scope Readiness

- The execution scope is now concrete and user-driven.
- The requested catalog and layout changes are small enough to fit in a tight iteration.
- The GitHub Actions panel is intentionally constrained to a safe first increment.

### Architecture Readiness

- Target repo already exists and is a working React/Vite application.
- The portal already has theme context, test scaffolding, and a dashboard composition to extend.
- The implementation bundle does not require backend or deployment-model changes.

### Repo and Environment Readiness

- Target repo: `TargetProjects/terminus/portal/terminus-portal`
- Existing tests and build pipeline are present
- The portal changes needed for portal3 map directly onto current source files in `src/`

### Verification Readiness

- `npm test` is the primary regression gate
- `npm run build` validates production bundling
- Story-level changes are concentrated in service config, dashboard layout, and one new Actions panel

## Risks

| Risk | Severity | Mitigation |
|---|---|---|
| GitHub Actions browser access may be incomplete for private repos | Medium | Keep the first implementation public-data only and document the limitation |
| Direct-IP Proxmox access may still surface certificate trust friction | Medium | Accept explicitly as an operator tradeoff, not an application error |
| CI/CD visibility requests may quickly expand to ArgoCD or release telemetry | Medium | Keep portal3’s first CI/CD slice bounded to Actions only |
| Domain/service grouping could break old tests and assumptions | Low | Update tests with the new grouping contract during implementation |

## Readiness Decision

**Gate Status:** ✅ PASSED

Portal3 is ready for `/dev`. The work is bounded, the target repo already supports the needed change pattern, and the major risks are known rather than hidden.
