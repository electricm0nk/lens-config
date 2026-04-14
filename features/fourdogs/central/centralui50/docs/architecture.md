# Architecture: fourdogs-central-centralui50

## 1. Scope

Technical plan to recover Central UI styling, theme behavior, Kaylee panel rendering, and ordering interaction reliability.

## 2. Proposed Architecture Work

### A. Frontend Rendering Reliability

- Validate root app shell initializes design-system providers in all auth routes.
- Ensure global CSS and token layers are loaded before route rendering.
- Replace any residual placeholder-page logic with real empty-state components.

### B. Theme System Restoration

- Introduce or repair theme provider wiring at app root.
- Persist user preference in local storage with safe defaults.
- Ensure body/html class switching updates all route layouts.

### C. Kaylee Panel Restoration

- Re-enable split-pane chair-phase layout.
- Restore Kaylee panel mount conditions and API data dependencies.
- Add resilient fallback UI when Kaylee backend is unavailable.

### D. Ordering Flow Restoration

- Verify API contract bindings for order read/write paths.
- Repair disabled/hidden mutation actions.
- Add client-side error surfaces for failed order mutations.

### E. Deployment Safety Net

- Validate UI assets exist and are non-placeholder before image push.
- Keep source/reference linkage explicit in workflow metadata.
- Fail fast on missing bundle markers (for example, missing built asset manifest).

## 3. Validation Strategy

- Route-level smoke checks for dashboard and order detail.
- Theme toggling check in both authenticated shell and order view.
- Kaylee panel presence + interaction smoke check.
- Order flow smoke check: create, edit quantity, submit.
- CI checks for asset integrity and workflow branch/ref correctness.

## 4. Rollout Plan

1. Ship UI reliability and theme fixes.
2. Ship Kaylee/order-flow fixes.
3. Ship CI/deploy safeguards.
4. Run dev and prod smoke verification after deployment.
