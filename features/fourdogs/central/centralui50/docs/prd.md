# Product Requirements Document: fourdogs-central-centralui50

## 1. Objective

Deliver a stable, modern Central UI experience that restores expected styling and all critical ordering interactions.

## 2. User Pain to Resolve

- Post-login interface appears unstyled and visually regressed.
- Missing theme experience (light/dark).
- Missing Kaylee workflow panel.
- Inability to place orders.

## 3. Functional Requirements

1. Authenticated UI shell must load full design system styles on every route.
2. Theme system must support light/dark mode with persisted preference.
3. Order detail page must render:
   - Floor Walk section
   - Chair phase section
   - Kaylee interaction panel
4. Order actions must work end-to-end:
   - create order
   - edit quantities
   - submit/mark submitted
5. Empty states must be intentional and styled, not placeholder artifacts.
6. Build/deploy pipeline must verify UI asset integrity before release.

## 4. Non-Functional Requirements

1. First meaningful paint for dashboard under normal internal network conditions: <= 3s.
2. No blocking JS/runtime errors on primary routes.
3. UI deploy path must be deterministic and traceable to UI source reference.
4. Accessibility baseline for interactive controls (keyboard focus, contrast-safe text in both themes).

## 5. Acceptance Criteria

1. Login lands on styled dashboard with no placeholder text.
2. Theme toggle works and persists after refresh.
3. Kaylee panel is visible and interactive in chair phase.
4. At least one complete manual smoke test passes: create order -> adjust items -> submit.
5. Release workflow rejects build if UI bundle integrity checks fail.

## 6. Risks

- Regression reintroduced by pipeline overwriting embedded assets.
- Drift between central-ui source branch and deploy target branch.
- Hidden frontend runtime errors from stale API contract assumptions.

## 7. Dependencies

- fourdogs-central-ui frontend repo
- fourdogs-central release workflows
- Existing API endpoints for orders/Kaylee/session
