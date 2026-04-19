---
feature: portal3
doc_type: epics
status: draft
goal: "Define the execution bundle for portal3 as a focused portal iteration covering service-catalog correctness, domain/service dashboard layout, and first-pass GitHub Actions visibility."
key_decisions:
  - "Use the current React/Vite portal as the execution base rather than a rewrite."
  - "Treat the user-requested service and layout fixes as the first implementation epic."
  - "Limit CI/CD visibility to a safe GitHub Actions first slice; defer authenticated or deployment-level telemetry."
open_questions:
  - "Should the Actions panel eventually include private repos through a proxy-backed feed?"
  - "Should ArgoCD sync/deploy state be a separate follow-up epic?"
depends_on: []
blocks: []
updated_at: 2026-04-19T02:18:00Z
---

# Epics — Terminus Portal portal3

**Initiative:** `terminus-portal-portal3`
**Track:** `express`
**Date:** 2026-04-19

---

## Overview

Portal3 is an iteration on the existing Terminus portal. The execution bundle turns the current planning set into a short, delivery-focused sequence:

- correct the live service inventory
- reorganize the dashboard around domain and service boundaries
- reduce wasted space on wide screens
- add a safe first pass at CI/CD visibility through GitHub Actions
- lock in the changes with automated verification

## Requirements Inventory

### Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | Proxmox card points to the direct IP address `https://10.0.0.48:8006` and uses the corresponding API health endpoint |
| FR2 | Consul and pgAdmin are removed from the portal catalog |
| FR3 | Kaylee Agent is displayed in the AI section |
| FR4 | Service cards are grouped by Domain and Service instead of broad category-only sections |
| FR5 | Dashboard layout is denser on wide screens to reduce unnecessary scrolling |
| FR6 | A GitHub Actions visibility panel shows current workflow activity for selected repos |
| FR7 | The Actions panel supports refresh, empty, and error states without breaking the rest of the dashboard |
| FR8 | Services without vendor icons still render with a stable fallback visual |

### Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR1 | No browser-side secrets or PATs may be embedded in the portal code |
| NFR2 | The GitHub Actions slice must work safely in a static frontend for public workflow data |
| NFR3 | The dashboard remains usable on both wide desktop layouts and narrower screens |
| NFR4 | Existing automated tests and build checks continue to pass after portal3 changes |
| NFR5 | Portal3 keeps the current React/Vite and static deployment model |

### Technical Requirements

| ID | Requirement |
|---|---|
| TR1 | `src/config/services.js` remains the source of truth for the service catalog |
| TR2 | The GitHub Actions panel is driven by explicit repo config, not hardcoded inline placeholders |
| TR3 | Components may only consume theme tokens that actually exist in the active theme objects |
| TR4 | Layout changes must degrade gracefully below the wide-screen breakpoint |

## FR Coverage Map

| FR/NFR | Epic |
|---|---|
| FR1, FR2, FR3, TR1 | E1 |
| FR4, FR5, FR8, NFR3, TR4 | E2 |
| FR6, FR7, NFR1, NFR2, TR2 | E3 |
| NFR4, TR3 | E4 |
| NFR5 | E1, E2, E3, E4 |

## Epic List

| Epic | Title | Depends On |
|---|---|---|
| E1 | Service Catalog Corrections | — |
| E2 | Domain/Service Dashboard Layout | E1 |
| E3 | GitHub Actions Visibility | E2 |
| E4 | Verification and Polish | E1, E2, E3 |

## Epic 1: Service Catalog Corrections

**Goal:** Bring the live portal inventory in line with real operator availability and endpoint reality.

**Requirements:** FR1, FR2, FR3, TR1

### Stories

| Story | Title |
|---|---|
| 1.1 | Correct portal service inventory and endpoints |
| 1.2 | Normalize service metadata for domain/service-aware rendering |

## Epic 2: Domain/Service Dashboard Layout

**Goal:** Replace broad category-only grouping with a denser, more intentional dashboard layout that uses wide screens effectively.

**Requirements:** FR4, FR5, FR8, NFR3, TR4

### Stories

| Story | Title |
|---|---|
| 2.1 | Group services by domain and service |
| 2.2 | Densify card layout and responsive dashboard composition |

## Epic 3: GitHub Actions Visibility

**Goal:** Add a first-pass CI/CD status surface that is useful immediately and safe for a static frontend.

**Requirements:** FR6, FR7, NFR1, NFR2, TR2

### Stories

| Story | Title |
|---|---|
| 3.1 | Configure monitored GitHub repositories and workflow-run summaries |
| 3.2 | Integrate the GitHub Actions HUD into the dashboard shell |

## Epic 4: Verification and Polish

**Goal:** Remove token/layout regressions and leave portal3 ready for implementation and rollout confidence.

**Requirements:** NFR4, TR3

### Stories

| Story | Title |
|---|---|
| 4.1 | Fix token-contract and fallback visual issues |
| 4.2 | Rebaseline tests and build verification for portal3 |
