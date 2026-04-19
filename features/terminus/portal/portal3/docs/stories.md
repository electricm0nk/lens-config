---
feature: portal3
doc_type: stories
status: draft
goal: "Break portal3 into independently deliverable stories that map directly to the user-requested fixes and the first CI/CD visibility increment."
key_decisions:
  - "Keep the story count small enough to execute as a focused iteration."
  - "Use one explicit story for GitHub Actions safety constraints so the implementation does not regress into client-side secret leakage."
open_questions:
  - "Should private-repo access be a future story or a separate initiative?"
depends_on: []
blocks: []
updated_at: 2026-04-19T02:18:00Z
---

# Stories — Terminus Portal portal3

**Initiative:** `terminus-portal-portal3`
**Track:** `express`
**Total Stories:** 8 across 4 epics

---

## Story Index

| Story | Title | Epic | Depends On |
|---|---|---|---|
| 1.1 | Correct portal service inventory and endpoints | E1 | — |
| 1.2 | Normalize service metadata for domain/service-aware rendering | E1 | 1.1 |
| 2.1 | Group services by domain and service | E2 | 1.2 |
| 2.2 | Densify card layout and responsive dashboard composition | E2 | 2.1 |
| 3.1 | Configure monitored GitHub repositories and workflow-run summaries | E3 | 2.2 |
| 3.2 | Integrate the GitHub Actions HUD into the dashboard shell | E3 | 3.1 |
| 4.1 | Fix token-contract and fallback visual issues | E4 | 2.2 |
| 4.2 | Rebaseline tests and build verification for portal3 | E4 | 3.2, 4.1 |

---

## Story 1.1: Correct Portal Service Inventory and Endpoints

**Epic:** E1 — Service Catalog Corrections  
**Depends On:** —  
**Estimated Size:** S

As a platform operator,  
I want the portal inventory to reflect currently reachable services and real endpoints,  
so that the dashboard does not send me to dead or misleading destinations.

**Acceptance Criteria:**

1. The Proxmox card links to `https://10.0.0.48:8006`
2. The Proxmox health check uses `https://10.0.0.48:8006/api2/json/version`
3. Consul is removed from the service catalog
4. pgAdmin is removed from the service catalog
5. Automated tests that assume the old service inventory are updated accordingly

---

## Story 1.2: Normalize Service Metadata for Domain/Service-Aware Rendering

**Epic:** E1 — Service Catalog Corrections  
**Depends On:** 1.1  
**Estimated Size:** S

As a developer,  
I want each service entry to include domain and service metadata,  
so that the dashboard can render a more intentional grouping model.

**Acceptance Criteria:**

1. Service entries include `domain` and `service` metadata
2. Kaylee Agent is classified under the AI service grouping
3. Existing visible services render correctly after the metadata expansion
4. Missing icon slugs degrade gracefully in the UI

---

## Story 2.1: Group Services by Domain and Service

**Epic:** E2 — Domain/Service Dashboard Layout  
**Depends On:** 1.2  
**Estimated Size:** M

As an operator,  
I want services grouped by Domain and Service,  
so that the portal matches how I think about the environment rather than forcing category scanning.

**Acceptance Criteria:**

1. Services render in grouped sections based on `domain` and `service`
2. Terminus Platform and Terminus AI appear as distinct sections when both have services
3. Fourdogs Central renders as its own section when present
4. Services retain stable ordering within each section
5. Grouping tests assert the new domain/service structure

---

## Story 2.2: Densify Card Layout and Responsive Dashboard Composition

**Epic:** E2 — Domain/Service Dashboard Layout  
**Depends On:** 2.1  
**Estimated Size:** M

As a wide-screen user,  
I want the dashboard to use the available horizontal space better,  
so that I do not need to scroll through excessive vertical padding.

**Acceptance Criteria:**

1. Service groups render in a denser grid layout on wide screens
2. The dashboard shell uses a multi-column composition when space allows
3. The layout collapses to a single-column composition at narrower widths
4. Metrics and auxiliary panels remain readable after the layout change

---

## Story 3.1: Configure Monitored GitHub Repositories and Workflow-Run Summaries

**Epic:** E3 — GitHub Actions Visibility  
**Depends On:** 2.2  
**Estimated Size:** M

As an operator,  
I want to see GitHub Actions activity for the most relevant repos,  
so that I can get first-pass CI/CD visibility from the portal.

**Acceptance Criteria:**

1. The monitored repo list is defined in a dedicated config module
2. The first monitored repos include `electricm0nk/terminus-portal`, `electricm0nk/terminus.infra`, and `electricm0nk/lens.config`
3. The implementation does not embed a GitHub PAT or other secret in client code
4. The panel summarizes queued, running, failed, and recent successful workflow runs using public API data only
5. The panel handles fetch errors without crashing the app

---

## Story 3.2: Integrate the GitHub Actions HUD into the Dashboard Shell

**Epic:** E3 — GitHub Actions Visibility  
**Depends On:** 3.1  
**Estimated Size:** S

As an operator,  
I want the Actions HUD visible alongside the main dashboard content,  
so that pipeline state is easy to scan during normal portal usage.

**Acceptance Criteria:**

1. The Actions panel is rendered in the dashboard shell
2. It supports manual refresh
3. It shows repo cards plus active workflow rows
4. It remains visually coherent with the current portal themes
5. It clearly communicates that private-repo or runner-level visibility will require a later proxy-backed implementation

---

## Story 4.1: Fix Token-Contract and Fallback Visual Issues

**Epic:** E4 — Verification and Polish  
**Depends On:** 2.2  
**Estimated Size:** S

As a developer,  
I want component styling to rely only on valid theme tokens and resilient fallbacks,  
so that portal3 does not introduce avoidable runtime UI regressions.

**Acceptance Criteria:**

1. Components no longer consume undefined theme tokens
2. The metrics panel uses a valid theme token set
3. Fallback service visuals render consistently when vendor icons are missing
4. Existing component tests continue to pass with the updated visuals

---

## Story 4.2: Rebaseline Tests and Build Verification for portal3

**Epic:** E4 — Verification and Polish  
**Depends On:** 3.2, 4.1  
**Estimated Size:** S

As a developer,  
I want the updated portal behavior verified by the current test and build pipeline,  
so that the portal3 changes are safe to merge and release.

**Acceptance Criteria:**

1. `npm test` passes in the portal repo
2. `npm run build` passes in the portal repo
3. Updated tests cover the new grouping model
4. FinalizePlan artifacts reflect the verified implementation-ready scope
