# Story 5.3: Theme Toggle and Refresh in Header

Status: done

## Story

As a user,
I want the theme toggle and manual refresh controls in the header,
so that both controls are immediately accessible without scrolling.

## Acceptance Criteria

1. Theme toggle button in Header calls `toggleTheme()` from `useTheme()` on click
2. Refresh button in Header calls `onRefresh` prop on click
3. Both buttons have correct `aria-label` as specified in Stories 2.3 and 4.4
4. Button styles fully token-driven
5. Component tests: toggle calls toggleTheme, refresh calls onRefresh, aria-labels correct

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #5)
  - [ ] `src/__tests__/components/Header.controls.test.jsx`:
    - [ ] Theme toggle button present with aria-label "Switch to Terminal theme" (when spaceops active)
    - [ ] Refresh button present with aria-label "Refresh health status"
    - [ ] Clicking toggle → `toggleTheme()` called (mock `useTheme`)
    - [ ] Clicking refresh → `onRefresh` prop called
    - [ ] Refresh button disabled when `isPolling === true`
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Move `ThemeToggle` into `Header.jsx` (AC: #1, #3, #4)
  - [ ] Import and render `<ThemeToggle />` (from Story 2.3) in Header right-side controls area
  - [ ] Remove standalone ThemeToggle from App.jsx (it now lives in Header)
- [ ] Task 3: Move `RefreshButton` into `Header.jsx` (AC: #2, #3, #4)
  - [ ] Add `onRefresh` and `isPolling` props to `Header`
  - [ ] Render `<RefreshButton onRefresh={onRefresh} isPolling={isPolling} />` in Header controls
  - [ ] Remove standalone RefreshButton from App.jsx position
  - [ ] Update App.jsx: pass `onRefresh={handleRefresh}` and `isPolling={isPolling}` to Header
- [ ] Task 4: Run all tests — green (AC: #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 2.3 (`ThemeToggle.jsx`), Story 4.4 (`RefreshButton.jsx`), Story 5.2 (`Header.jsx` with summary props).
**This story completes the Header** — all content, summary, and controls are now in place.
**Clean up App.jsx:** After this story, `App.jsx` should no longer render standalone ThemeToggle or RefreshButton. Both are embedded in Header.
**Props flow after this story:**
```
App
  → Header:
      onRefresh={handleRefresh}
      isPolling={isPolling}
      onlineCount + totalCount + unreachableCount + lastChecked    (from Story 5.2)
    Header renders:
      ThemeToggle (calls toggleTheme internally via useTheme)
      RefreshButton (calls onRefresh, disabled when isPolling)
```

### Project Structure Notes

Modified:
- `src/components/Header.jsx` (add ThemeToggle + RefreshButton, accept onRefresh + isPolling props)
- `src/App.jsx` (remove standalone ThemeToggle + RefreshButton, pass onRefresh + isPolling to Header)

New tests:
```
src/
└── __tests__/components/
    └── Header.controls.test.jsx
```

No new standalone component files — ThemeToggle and RefreshButton already exist.

### References

- FR3 (header controls): `docs/terminus/portal/portal2/epics.md`
- Story 2.3 (ThemeToggle — aria-label spec)
- Story 4.4 (RefreshButton — aria-label spec, isPolling prop)
- Story 5.1/5.2 (Header component baseline)
- Story 7.2 (aria-labels audited in E2E)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
