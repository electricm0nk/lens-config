---
initiative: terminus-portal-portal2
track: tech-change
phase: devproposal
date: "2026-04-13"
reviewer: Perturabo (Architect) + John (PM)
---

# Implementation Readiness Checklist — Terminus Platform Portal (portal2)

**Initiative:** `terminus-portal-portal2`
**Track:** `tech-change`
**Assessment Date:** 2026-04-13

---

## Summary

| Category | Status |
|---|---|
| Architecture | ✅ READY |
| Requirements | ✅ READY |
| Stories | ✅ READY |
| Infrastructure | ✅ READY |
| Risks | ⚠️ NOTED (no blockers) |
| **Overall** | **✅ PROCEED** |

---

## 1. Architecture Readiness

| Check | Status | Notes |
|---|---|---|
| Architecture document exists | ✅ | `docs/terminus/portal/portal2/architecture.md` — 574 lines |
| All components defined | ✅ | Full `src/` directory tree, responsibilities documented |
| Technology decisions made | ✅ | React/Vite, ThemeContext, static services.js, UseHealthCheck hook |
| Data flow documented | ✅ | Props-down/events-up clearly described for all component relationships |
| No-cors health check constraint documented | ✅ | Binary ONLINE/UNREACHABLE; VAULT_STATUS_DETAIL deferred |
| Existing infrastructure leverage confirmed | ✅ | Existing Helm chart at `deploy/helm/` adapted (not replaced) |
| Security: no secrets in frontend | ✅ | All URLs static config — no tokens, passwords, or API keys in frontend |
| Deployment target confirmed | ✅ | k3s, namespace `terminus-portal`, ArgoCD, `ghcr.io/electricm0nk/terminus-portal` |

---

## 2. Requirements Readiness

| Check | Status | Notes |
|---|---|---|
| All FRs traced to epics | ✅ | FR1–FR9 coverage map in epics.md |
| All NFRs traced to epics | ✅ | NFR1–NFR6 coverage map in epics.md |
| All TRs traced to stories | ✅ | TR1–TR7 coverage map in epics.md |
| Consul added to service registry | ✅ | Added during devproposal (FR1 updated, 9th service) |
| No open requirements ambiguities | ✅ | All decisions recorded in architecture.md |
| Vault binary health check rationale documented | ✅ | no-cors opaque constraint; `VAULT_STATUS_DETAIL` wiring point noted |

---

## 3. Stories Readiness

| Check | Status | Notes |
|---|---|---|
| All stories have Given/When/Then ACs | ✅ | 18 stories, all with structured ACs |
| Stories are independently deliverable | ✅ | Dependency chain documented in stories.md |
| E1 stories are unblocked (first sprint) | ✅ | All 4 E1 stories can start on day 1 sequentially |
| E6 identified as parallel track | ✅ | Can be worked in parallel with E2–E5 after E1 |
| E7 identified as final phase | ✅ | Requires all components exist; Playwright + axe-core |
| Story sizes estimated | ✅ | S/M assigned to each story in stories.md |
| Fourdogs enablement deferred | ✅ | One-line change in `services.js` — no sprint story needed |
| Metric wiring deferred | ✅ | E6 stories complete as stub; upgrade path in architecture.md |

---

## 4. Infrastructure Readiness

| Check | Status | Notes |
|---|---|---|
| Target repo exists | ✅ | `https://github.com/electricm0nk/terminus-portal` confirmed |
| Existing Helm chart inspected | ✅ | `deploy/helm/` present and working |
| Current deployment confirmed live | ✅ | `https://portal.trantor.internal` serving static HTML v1 |
| Container registry accessible | ✅ | `ghcr.io/electricm0nk/terminus-portal` in use |
| Local repo clone available | ✅ | `TargetProjects/terminus/portal/terminus-portal` |
| ArgoCD reconciliation path known | ✅ | Namespace `terminus-portal`, existing ArgoCD app |

---

## 5. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| no-cors opaque health responses limit Vault status granularity | LOW | Binary ONLINE/UNREACHABLE documented; `VAULT_STATUS_DETAIL` wiring point reserved |
| Simple Icons CDN availability | LOW | All icons from single CDN; acceptable for internal tool |
| Fourdogs service not yet deployed | LOW | `enabled: false` in SERVICES; no polling; trivial enablement when ready |
| E7 Playwright + axe-core E2E setup time | MEDIUM | Allocated as dedicated stories (7.1, 7.2) with realistic sizing |
| CSS-in-JS inline style performance at scale | LOW | Only 15 cards; no performance concern |

---

## 6. Open Items (Non-Blocking)

| Item | Owner | Resolution |
|---|---|---|
| `VAULT_STATUS_DETAIL` — Vault proxy tier for detailed status breakdown | Future initiative | Wiring point documented in architecture.md; no action for portal2 |
| Metric data wiring (Prometheus/InfluxDB) | Future initiative | E6 creates stub panel; upgrade path documented in architecture.md |
| Fourdogs app card enablement | Platform team | One-line change when fourdogs is deployed |

---

## Readiness Decision

**Gate Status:** ✅ PASSED — No blockers. All required artifacts exist, all decisions made, all stories are implementation-ready.

**Conditions:** None. Proceed to sprint planning.
