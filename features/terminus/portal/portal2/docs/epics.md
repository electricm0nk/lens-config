---
stepsCompleted: [1, 2, 3]
inputDocuments:
  - "docs/terminus/portal/portal2/architecture.md"
workflowType: epics-and-stories
project_name: terminus-portal-portal2
initiative_root: terminus-portal-portal2
track: tech-change
date: "2026-04-13"
---

# Epics & Stories — Terminus Platform Portal (portal2)

**Initiative:** `terminus-portal-portal2`
**Track:** `tech-change`
**Date:** 2026-04-13

---

## Overview

React SPA replacement for the static HTML Terminus portal (`v1`). Serves as the primary Terminus Platform launchpad — 9 service cards with live health polling, 6 metric stub cards, dual-theme system (Terminal / Space Ops), deployed on k3s via existing Helm chart.

---

## Requirements Inventory

### Functional Requirements

| ID | Requirement |
|---|---|
| FR1 | Service registry — 9 services (ArgoCD, Vault, Semaphore, k3s Dashboard, pgAdmin, Ollama Gateway, Proxmox, Consul, Fourdogs) with name, description, category, URL, icon, health check config, enabled flag |
| FR2 | Health polling — every 30s, 5s timeout, fetch no-cors, binary ONLINE/UNREACHABLE status |
| FR3 | Header — cluster identity, X/Y ONLINE summary, last-checked timestamp, theme toggle, manual refresh button |
| FR4 | Metric stub panel — 6 cards (CPU Load, Memory Used, Net In, Net Out, Pods Running, Disk Used), all unwired, source badge, `—` value, "WIRE TO [SOURCE] TO ENABLE" label |
| FR5 | Theme system — Terminal (dark green/CRT scanlines) and Space Ops (navy/HUD), persisted to localStorage |
| FR6 | Disabled service card — fourdogs renders as non-interactive, no health polling |
| FR7 | Status indicators — CHECKING/ONLINE/UNREACHABLE/NO_CHECK with aria-labels |
| FR8 | Manual refresh — forces immediate re-poll of all health checks |
| FR9 | Simple Icons CDN for service icons with accessible alt text |

### Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR1 | TDD — tests written before implementation; full pyramid: unit, component, integration, E2E |
| NFR2 | Static build — `dist/` output, no backend, no SSR |
| NFR3 | k3s deployment — existing Helm chart at `deploy/helm/`, ArgoCD reconciled, namespace `terminus-portal` |
| NFR4 | Multi-stage Dockerfile — node:20-alpine build + nginx:1.27-alpine serve |
| NFR5 | Accessibility — semantic HTML, keyboard navigable, screen reader labels on all interactive and status elements |
| NFR6 | No secrets — all service URLs are static config in `src/config/services.js` |

### Additional Technical Requirements

| ID | Source | Requirement |
|---|---|---|
| TR1 | Architecture | `ThemeContext` provider with `useTheme()` hook; token objects in `src/themes/`; no hardcoded hex values in components |
| TR2 | Architecture | `src/config/services.js` exports `SERVICES` array; `src/config/metrics.js` exports `METRICS` array |
| TR3 | Architecture | `STATUS` enum: CHECKING, ONLINE, UNREACHABLE, NO_CHECK |
| TR4 | Architecture | Health check uses `AbortController` for timeout enforcement |
| TR5 | Architecture | Vault health check is binary (VAULT_STATUS_DETAIL deferred — no-cors opaque constraint) |
| TR6 | Architecture | nginx.conf: `try_files $uri $uri/ /index.html` for SPA routing; `server_tokens off` |
| TR7 | Architecture | Metric prop shape: `{ id, label, unit, source, value, wired }` |

---

## FR Coverage Map

| FR/NFR | Epic |
|---|---|
| NFR1, NFR2, NFR3, NFR4, TR6 | E1 |
| FR5, TR1 | E2 |
| FR1, FR6, FR9, TR2 | E3 |
| FR2, FR7, FR8, TR3, TR4, TR5 | E4 |
| FR3 | E5 |
| FR4, TR7 | E6 |
| NFR5 | E7 |
| NFR6 | E1, E3 |
| NFR3 (deploy) | E8 |

---

## Epic List

| Epic | Title | Depends On |
|---|---|---|
| E1 | Foundation & Build Pipeline | — |
| E2 | Theme System | E1 |
| E3 | Service Registry & Navigation | E2 |
| E4 | Health Polling & Status Display | E3 |
| E5 | Header & Live Summary | E4 |
| E6 | Metrics Stub Panel | E1 |
| E7 | Accessibility Hardening | E5, E6 |
| E8 | Deployment | E7 |

---

## Epic 1: Foundation & Build Pipeline

**Goal:** Scaffold the full Vite/React project, wire CI, adapt the Helm chart, and deploy a working empty shell to k3s. Every subsequent epic builds on this foundation.

**Requirements:** NFR1, NFR2, NFR3, NFR4, NFR6, TR6

---

### Story 1.1: Vite/React Project Scaffold with TDD Baseline

As a developer,
I want a Vite/React project initialized with Vitest and React Testing Library,
So that every subsequent story has a runnable test suite before implementation begins.

**Acceptance Criteria:**

**Given** the `terminus-portal` repository on a feature branch
**When** the developer runs `npm install && npm test`
**Then** all tests pass (baseline pass with zero test files is acceptable at this stage)
**And** running `npm run build` produces a `dist/` folder with an `index.html`
**And** no hardcoded hex color values exist anywhere in `src/`
**And** `src/config/services.js` exports an empty `SERVICES` array placeholder
**And** `src/config/metrics.js` exports an empty `METRICS` array placeholder
**And** `src/constants/status.js` exports `STATUS = { CHECKING, ONLINE, UNREACHABLE, NO_CHECK }`

---

### Story 1.2: Multi-Stage Dockerfile and nginx.conf

As a platform operator,
I want a multi-stage Docker build that produces a minimal nginx-served image,
So that the portal can be built and deployed without Node.js in the runtime container.

**Acceptance Criteria:**

**Given** a clean Docker build context
**When** `docker build -t terminus-portal:test .` completes
**Then** the image is based on `nginx:1.27-alpine` in the final stage
**And** `docker run --rm -p 8080:80 terminus-portal:test` serves the app at `http://localhost:8080`
**And** `curl -s http://localhost:8080/nonexistent` returns a 200 response (SPA fallback — `index.html`)
**And** the response headers do not include `Server: nginx` or version info (`server_tokens off`)
**And** static assets (`.js`, `.css`) return `Cache-Control: public, immutable`

---

### Story 1.3: Helm Chart Adaptation for Vite Build

As a platform operator,
I want the existing Helm chart at `deploy/helm/` adapted for the React/Vite image,
So that ArgoCD can deploy the new portal image without chart restructuring.

**Acceptance Criteria:**

**Given** the updated `deploy/helm/values.yaml`
**When** `helm template terminus-portal deploy/helm/` is run
**Then** the output includes a Deployment referencing `ghcr.io/electricm0nk/terminus-portal`
**And** the Ingress resource specifies host `portal.trantor.internal`
**And** the namespace is `terminus-portal`
**And** no Docker Compose files exist anywhere in the repository
**And** a test (`helm lint deploy/helm/`) passes with no errors or warnings

---

### Story 1.4: CI Pipeline — Build, Test, Push

As a developer,
I want a GitHub Actions workflow that builds, tests, and pushes the image on merge to main,
So that ArgoCD always pulls a tested, tagged image.

**Acceptance Criteria:**

**Given** a push or PR merge to the `main` branch
**When** the GitHub Actions workflow runs
**Then** `npm ci && npm test` must pass before Docker build proceeds
**And** the Docker image is tagged with the git SHA and pushed to `ghcr.io/electricm0nk/terminus-portal`
**And** a `latest` tag is also updated on successful main merge
**And** branch protection prevents merge if tests fail
**And** the workflow file lives at `.github/workflows/ci.yml`

---

## Epic 2: Theme System

**Goal:** Implement the full theme architecture — context provider, token objects for both themes, localStorage persistence, and CRT scanline overlay — before any content components are built. Every subsequent component consumes `useTheme()`.

**Requirements:** FR5, TR1

---

### Story 2.1: Theme Token Objects and Context Provider

As a developer,
I want `ThemeContext` with Terminal and Space Ops token objects and a `useTheme()` hook,
So that every component can access typed tokens without prop drilling or hardcoded values.

**Acceptance Criteria:**

**Given** `ThemeProvider` wraps the application at the root
**When** a component calls `useTheme()`
**Then** it receives `{ tokens, themeName, toggleTheme }` where `tokens` is the full token object for the active theme
**And** `tokens` includes: `bg`, `bgSurface`, `bgCard`, `text`, `textMuted`, `accent`, `accentDim`, `border`, `borderActive`, `statusOnline`, `statusUnreachable`, `statusChecking`, `statusNoCheck`, `fontFamily`, `scanlines`
**And** no component file contains a hardcoded hex color value
**And** unit tests verify that `terminal` and `spaceops` token objects contain all required keys
**And** unit tests verify `useTheme()` throws outside of `ThemeProvider`

---

### Story 2.2: localStorage Theme Persistence

As a user,
I want my theme choice persisted across browser sessions,
So that I don't have to re-select my preferred theme on every visit.

**Acceptance Criteria:**

**Given** the user has selected the Terminal theme
**When** they close and reopen the portal in the same browser
**Then** the Terminal theme is active without any interaction
**And** if `localStorage` contains an invalid/unknown theme key, the default (`spaceops`) is used
**And** `localStorage.setItem` is called with key `terminus-portal-theme` on every `toggleTheme()` call
**And** unit tests cover: fresh load (no key → default), valid key → correct theme, invalid key → default

---

### Story 2.3: Theme Toggle and CRT Scanline Overlay

As a user,
I want to toggle between Terminal and Space Ops themes with a button,
So that I can switch between the dark-green CRT aesthetic and the HUD aesthetic.

**Acceptance Criteria:**

**Given** the portal is loaded
**When** the user clicks the theme toggle button
**Then** all themed elements immediately update to the new theme's token values without a page reload
**And** when the active theme is Terminal (`tokens.scanlines === true`), a fixed-position scanline overlay div is rendered over the page
**And** when the active theme is Space Ops (`tokens.scanlines === false`), the scanline overlay is not rendered
**And** `aria-label` on the toggle button reads "Switch to Terminal theme" or "Switch to Space Ops theme" based on current theme
**And** component tests verify the overlay renders/disappears on theme toggle

---

## Epic 3: Service Registry & Navigation

**Goal:** Render all 9 service cards from static config — with category grouping, icons, descriptions, and proper disabled state — without health data. The portal is usable as a simple launchpad after this epic.

**Requirements:** FR1, FR6, FR9, TR2, NFR6

---

### Story 3.1: Service Config and ServiceCard Component

As a platform operator,
I want all 9 services defined in `src/config/services.js` and rendered as individual cards,
So that the portal displays every service with its name, description, category, and URL.

**Acceptance Criteria:**

**Given** the `SERVICES` array in `src/config/services.js` contains all 9 entries
**When** the portal renders
**Then** exactly 9 `ServiceCard` components are rendered
**And** each card displays: service name, description, category label, and service icon from Simple Icons CDN
**And** each enabled card is an anchor tag (`<a>`) with `href` pointing to the service URL, `target="_blank"`, `rel="noopener noreferrer"`
**And** the Fourdogs card (`enabled: false`) renders as a non-interactive `<div>` (not an anchor) with a visual "pending" indicator
**And** icon `<img>` elements have non-empty `alt` attributes
**And** no service URL or health check URL is stored outside of `src/config/services.js`
**And** component tests cover: enabled card renders as link, disabled card renders as non-link, icon alt text present

---

### Story 3.2: ServiceGrid with Category Grouping

As a user,
I want services grouped by category with labeled sections,
So that I can quickly find a service by its operational domain.

**Acceptance Criteria:**

**Given** services with categories: `infra`, `platform`, `data`, `ai`, `app`
**When** `ServiceGrid` renders the `SERVICES` array
**Then** services are grouped by category with a visible category heading above each group
**And** services within a group are rendered in the order they appear in `SERVICES`
**And** a category section is only rendered if at least one service belongs to it
**And** component tests verify: correct grouping, correct heading per category, empty category not rendered

---

## Epic 4: Health Polling & Status Display

**Goal:** Layer live health status onto the service cards from Epic 3. Polling runs on a 30s interval with a 5s timeout, and the status indicators display CHECKING/ONLINE/UNREACHABLE/NO_CHECK.

**Requirements:** FR2, FR7, FR8, TR3, TR4, TR5

---

### Story 4.1: StatusIndicator Component

As a user,
I want a visual status indicator showing ONLINE, UNREACHABLE, CHECKING, or NO CHECK,
So that I can immediately see whether each service is reachable.

**Acceptance Criteria:**

**Given** a `StatusIndicator` receives a `status` prop from the `STATUS` enum
**When** it renders
**Then** it displays a colored dot using the correct theme token (`statusOnline`, `statusUnreachable`, `statusChecking`, `statusNoCheck`)
**And** it renders a text label matching the status value
**And** the root element has `aria-label` set to one of: "Status: Online", "Status: Unreachable", "Status: Checking", "Status: No health check"
**And** `role="status"` is present on the element
**And** unit tests cover all 4 status values for correct color token and aria-label

---

### Story 4.2: useHealthCheck Hook

As a developer,
I want a `useHealthCheck(url, enabled)` hook that polls a health endpoint,
So that health check logic is encapsulated and testable independently of components.

**Acceptance Criteria:**

**Given** `useHealthCheck` is called with a valid URL and `enabled: true`
**When** the fetch resolves (opaque or otherwise without network error)
**Then** the hook returns `STATUS.ONLINE`

**Given** the fetch throws a `TypeError` (network failure) or the `AbortController` fires after 5000ms
**When** the hook catches the error
**Then** the hook returns `STATUS.UNREACHABLE`

**Given** `useHealthCheck` is called with `enabled: false`
**When** it initializes
**Then** it returns `STATUS.NO_CHECK` immediately and never calls `fetch`

**And** the hook uses `AbortController` to enforce the 5s timeout
**And** the `AbortController` and interval are cleaned up on component unmount
**And** unit tests mock `fetch` to cover: opaque success → ONLINE, network error → UNREACHABLE, timeout → UNREACHABLE, disabled → NO_CHECK

---

### Story 4.3: App-Level Polling Loop and ServiceCard Status Integration

As a user,
I want each service card to display a live health status that updates every 30 seconds,
So that I can see the current reachability of all services at a glance.

**Acceptance Criteria:**

**Given** the portal has loaded and `SERVICES` contains services with `healthCheck.enabled: true`
**When** 30 seconds elapse after initial load
**Then** `checkHealth` is called for each service with `healthCheck.enabled: true`
**And** each `ServiceCard` receives the resolved `status` prop and renders the correct `StatusIndicator`
**And** services with `healthCheck.enabled: false` always show `STATUS.NO_CHECK` without triggering fetch
**And** `Promise.allSettled` is used so a single service failure does not block others
**And** integration tests mock `fetch` to return success for some services and error for others, then verify the correct status per service card

---

### Story 4.4: Manual Refresh Button

As a user,
I want a manual refresh button that immediately re-polls all health checks,
So that I can force an update without waiting for the 30-second interval.

**Acceptance Criteria:**

**Given** the portal is displaying stale health status
**When** the user clicks the refresh button
**Then** all services with `healthCheck.enabled: true` are re-polled immediately
**And** service cards transition to `STATUS.CHECKING` during the re-poll
**And** the refresh button is disabled while polling is in progress
**And** `aria-label` on the button is "Refresh health status"
**And** component tests verify: button disabled during poll, cards show CHECKING during poll, button re-enabled after all settled

---

## Epic 5: Header & Live Summary

**Goal:** Implement the full header with cluster identity, live X/Y ONLINE summary derived from the health state in App, last-checked timestamp, theme toggle button, and manual refresh button.

**Requirements:** FR3

---

### Story 5.1: Header Component with Cluster Identity

As a user,
I want a header displaying the platform name and cluster details,
So that I always know which environment the portal is connected to.

**Acceptance Criteria:**

**Given** the portal loads
**When** the `Header` renders
**Then** it displays: platform name ("Terminus Platform Portal"), cluster name ("trantor"), domain ("hintzmann.net"), and technology ("k3s")
**And** all text colors use theme tokens — no hardcoded hex values
**And** the header is rendered as a `<header>` element with `role="banner"`
**And** component tests verify: correct text rendered, all styles reference tokens

---

### Story 5.2: Live Health Summary and Last-Checked Timestamp

As a user,
I want the header to show how many services are online and when the last check ran,
So that I have an at-a-glance view of platform health without scanning every card.

**Acceptance Criteria:**

**Given** `App` has polled all services and resolved health state
**When** the `Header` renders with `onlineCount` and `totalCount` and `lastChecked` props
**Then** it displays "X/Y ONLINE" where X is services with `STATUS.ONLINE` and Y is services with `healthCheck.enabled: true`
**And** if any services are `STATUS.UNREACHABLE`, an error count badge is displayed ("N UNREACHABLE") in the `statusUnreachable` token color
**And** "Last checked: HH:MM:SS" is shown using the timestamp of the most recent completed poll
**And** during the initial load (all services still CHECKING), "Checking..." is displayed instead of X/Y
**And** unit tests cover: all online, some unreachable, all checking initial state

---

### Story 5.3: Theme Toggle and Refresh in Header

As a user,
I want the theme toggle and manual refresh controls in the header,
So that both controls are immediately accessible without scrolling.

**Acceptance Criteria:**

**Given** the header is rendered
**When** the user views it
**Then** the theme toggle button is present and calls `toggleTheme()` from `useTheme()` on click
**And** the refresh button is present and calls the `onRefresh` prop on click
**And** both buttons display the correct `aria-label` values as specified in Stories 2.3 and 4.4
**And** button styles are fully token-driven
**And** component tests verify: toggle calls toggleTheme, refresh calls onRefresh, aria-labels correct

---

## Epic 6: Metrics Stub Panel

**Goal:** Render the 6 metric stub cards from static config. All cards show `—` with source badge and "WIRE TO [SOURCE] TO ENABLE" label. Self-contained — no dependency on health polling or header.

**Requirements:** FR4, TR7

---

### Story 6.1: Metrics Config and MetricCard Component

As a developer,
I want all 6 metrics defined in `src/config/metrics.js` and a `MetricCard` component,
So that the metrics panel renders correctly and the wiring interface is locked in for future data feeds.

**Acceptance Criteria:**

**Given** `METRICS` in `src/config/metrics.js` contains exactly 6 entries (cpu-load, mem-used, net-in, net-out, pods-running, disk-used)
**When** `MetricCard` renders a metric with `wired: false`
**Then** it displays: the metric label, `—` as the value, the unit, a source badge ("PROMETHEUS" or "INFLUXDB"), and "WIRE TO [SOURCE] TO ENABLE" label
**And** `MetricCard` accepts a `metric` prop of shape `{ id, label, unit, source, value, wired }`
**And** when `wired: true` and `value` is a number, it renders the value and unit without the "WIRE TO" label (future-ready)
**And** all colors and fonts use theme tokens
**And** unit tests cover: unwired renders `—` + label, wired with value renders value, source badge displays correct source name

---

### Story 6.2: MetricsPanel Layout

As a user,
I want the 6 metric cards displayed in a clearly labeled panel,
So that the metrics section is visually distinct from the service cards.

**Acceptance Criteria:**

**Given** `MetricsPanel` receives the `METRICS` array
**When** it renders
**Then** exactly 6 `MetricCard` components are displayed
**And** the panel has a visible section heading ("Platform Metrics" or equivalent)
**And** cards are arranged in a responsive grid (2 columns on mobile, up to 3 on wider viewports — implemented via CSS grid with inline style token values)
**And** component tests verify: 6 cards rendered, heading present, each card receives correct metric prop

---

## Epic 7: Accessibility Hardening

**Goal:** Audit and remediate all interactive and status elements for keyboard navigation, screen reader labels, and semantic HTML. Produces a clean Playwright E2E pass.

**Requirements:** NFR5

---

### Story 7.1: Keyboard Navigation and Focus Management

As a keyboard user,
I want to navigate all interactive elements using Tab and Enter/Space,
So that the portal is fully usable without a mouse.

**Acceptance Criteria:**

**Given** the portal is loaded in a browser
**When** the user presses Tab repeatedly from the browser address bar
**Then** focus cycles through: theme toggle → refresh button → all enabled service cards (in DOM order)
**And** disabled service cards (fourdogs) are NOT in the tab order (`tabIndex=-1` or non-interactive element)
**And** pressing Enter or Space on a focused service card navigates to the service URL
**And** focus indicators are visible and use the `borderActive` token color (not the browser default outline removed)
**And** Playwright E2E test verifies the full tab cycle reaches all enabled cards

---

### Story 7.2: Screen Reader Labels and Semantic HTML Audit

As a screen reader user,
I want all status indicators, buttons, and icons to have meaningful labels,
So that the portal content is fully audible without visual context.

**Acceptance Criteria:**

**Given** the portal HTML is audited
**When** a screen reader traverses the page
**Then** every `StatusIndicator` has `aria-label` matching "Status: Online/Unreachable/Checking/No health check"
**And** every service card icon `<img>` has a non-empty `alt` attribute matching the service name
**And** the theme toggle button `aria-label` describes the action ("Switch to Terminal theme" / "Switch to Space Ops theme")
**And** the refresh button `aria-label` is "Refresh health status"
**And** the metrics section heading and each metric card label are readable by screen reader
**And** no `<div>` or `<span>` is used for interactive elements that should be `<button>` or `<a>`
**And** Playwright E2E test checks `aria-label` on status indicators for at least 3 services
**And** axe-core accessibility scan returns zero violations at WCAG 2.1 AA level

---

## Implementation Readiness Notes

- **E1 must be complete** before any other epic begins — it establishes the test runner, build pipeline, and config skeleton
- **E6 is fully parallel** with E2–E5 after E1
- **E7 is the final pass** — all components must exist before accessibility audit
- `VAULT_STATUS_DETAIL` wiring point: documented in `architecture.md`; no story required until proxy tier exists
- `fourdogs` enablement: one-line change in `services.js` + one PR; no story required
- Metric wiring: documented upgrade path in `architecture.md`; E6 stories are complete as written

---

## Epic 8: Deployment

**Goal:** Wire the full Temporal release pipeline for `terminus-portal`, deploy portal2 to the dev environment for verification, then promote to production — replacing the v1 static portal at `portal.trantor.internal`.

**Requirements:** NFR3 (k3s deployment via Temporal release contract)

---

### Story 8.1: Portal2 Deployment — Release Pipeline Wiring and Production Launch

As a platform operator,
I want `terminus-portal` (portal2) deployed to both dev and production k3s environments via the standard Temporal release pipeline,
So that every future portal change flows through the automated GitOps release contract (build → promote → Temporal → ArgoCD → smoke test).

**Acceptance Criteria:**

**Given** `release.yml` exists and `terminus.infra` is fully wired for both dev and prod
**When** a PR merges to `develop`
**Then** Temporal `ReleaseWorkflow` completes for the dev environment and portal2 is reachable at `https://portal-dev.trantor.internal`

**When** a PR then merges to `main`
**Then** Temporal `ReleaseWorkflow` completes for the prod environment and portal2 is reachable at `https://portal.trantor.internal`, replacing the v1 static portal

**And** all Temporal activities complete without failure: `SeedSecrets` → `TriggerArgoCDSync` → `WaitForArgoCDSync` → `RunSmokeTest`
**And** no `syncPolicy.automated` on either `terminus-portal` or `terminus-portal-dev` ArgoCD Applications
**And** Semaphore templates registered in project 3 (`terminus-infra`) — not project 2
**And** verify playbooks use `http://portal[-dev].trantor.internal` external DNS (NOT cluster-internal DNS)

**See:** `docs/terminus/portal/portal2/dev-stories/dev-story-8-1.md` for full task breakdown, file list, and cicd.md reference mapping.
