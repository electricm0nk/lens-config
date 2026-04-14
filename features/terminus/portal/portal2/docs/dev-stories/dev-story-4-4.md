# Story 4.4: Manual Refresh Button

Status: done

## Story

As a user,
I want a manual refresh button that immediately re-polls all health checks,
so that I can force an update without waiting for the 30-second interval.

## Acceptance Criteria

1. Clicking refresh button immediately re-polls all services with `healthCheck.enabled: true`
2. Service cards transition to `STATUS.CHECKING` during the re-poll
3. Refresh button is disabled while polling is in progress
4. `aria-label` on button is "Refresh health status"
5. Component tests: button disabled during poll, cards show CHECKING during poll, button re-enabled after all settled

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #5)
  - [ ] `src/__tests__/components/RefreshButton.test.jsx`:
    - [ ] Button has aria-label="Refresh health status"
    - [ ] Button is enabled when `isPolling === false`
    - [ ] Button is disabled when `isPolling === true`
    - [ ] Clicking button calls `onRefresh` prop
  - [ ] `src/__tests__/integration/refresh.test.jsx`:
    - [ ] While polling: service cards show `STATUS.CHECKING`
    - [ ] After poll completes: button re-enabled
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/components/RefreshButton.jsx` (AC: #3, #4)
  - [ ] Props: `{ onRefresh, isPolling }`
  - [ ] `<button onClick={onRefresh} disabled={isPolling} aria-label="Refresh health status">`
  - [ ] All styles from `useTheme()` tokens
  - [ ] Visual disabled state: reduced opacity using theme token
- [ ] Task 3: Update `App.jsx` to reset statusMap to CHECKING on refresh (AC: #2)
  - [ ] `handleRefresh` function:
    ```js
    const handleRefresh = () => {
      // Reset all enabled services to CHECKING
      const checkingMap = SERVICES.reduce((acc, s) => {
        acc[s.id] = s.healthCheck.enabled ? STATUS.CHECKING : STATUS.NO_CHECK;
        return acc;
      }, {});
      setStatusMap(checkingMap);
      checkAllServices();  // re-triggers polling
    };
    ```
  - [ ] Pass `onRefresh={handleRefresh}` and `isPolling={isPolling}` to RefreshButton
- [ ] Task 4: Render RefreshButton in `App.jsx` (temporary position — moves to Header in Story 5.3)
- [ ] Task 5: Run all tests — green (AC: #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 4.3 — `checkAllServices`, `isPolling`, and `statusMap` state exist in App.
**Note:** RefreshButton is placed temporarily near the ServiceGrid in App. It will be relocated into `Header` in Story 5.3. Keep the component self-contained (accepts `onRefresh` + `isPolling` as props).
**CHECKING transition (AC: #2):** Before calling `checkAllServices()`, explicitly reset all healthCheck-enabled services to `STATUS.CHECKING` in statusMap. This ensures the visual feedback is immediate.
**Disabled state accessibility:** Use `disabled` attribute on `<button>` — screen readers will announce "dimmed" or "greyed out". aria-label remains "Refresh health status" even when disabled.

### Project Structure Notes

New:
```
src/
├── components/
│   └── RefreshButton.jsx
└── __tests__/
    ├── components/
    │   └── RefreshButton.test.jsx
    └── integration/
        └── refresh.test.jsx
```
Modified:
- `src/App.jsx` (add handleRefresh, wire RefreshButton)

### References

- FR8 (manual refresh): `docs/terminus/portal/portal2/epics.md`
- FR7 (CHECKING status during refresh): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Health Polling"
- Story 5.3 (RefreshButton moves into Header)
- Story 7.2 (aria-label audited)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
