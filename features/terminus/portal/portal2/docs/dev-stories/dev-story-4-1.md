# Story 4.1: StatusIndicator Component

Status: done

## Story

As a user,
I want a visual status indicator showing ONLINE, UNREACHABLE, CHECKING, or NO CHECK,
so that I can immediately see whether each service is reachable.

## Acceptance Criteria

1. `StatusIndicator` receives `status` prop from `STATUS` enum
2. Displays colored dot using correct theme token (`statusOnline`, `statusUnreachable`, `statusChecking`, `statusNoCheck`)
3. Renders text label matching the status value
4. Root element has `aria-label`: "Status: Online", "Status: Unreachable", "Status: Checking", "Status: No health check"
5. `role="status"` present on the root element
6. Unit tests cover all 4 status values for correct color token and aria-label

## Tasks / Subtasks

- [ ] Task 1: Write failing unit tests FIRST (TDD) (AC: #6)
  - [ ] `src/__tests__/components/StatusIndicator.test.jsx`:
    - [ ] `STATUS.ONLINE` → dot uses `statusOnline` token, aria-label="Status: Online"
    - [ ] `STATUS.UNREACHABLE` → dot uses `statusUnreachable` token, aria-label="Status: Unreachable"
    - [ ] `STATUS.CHECKING` → dot uses `statusChecking` token, aria-label="Status: Checking"
    - [ ] `STATUS.NO_CHECK` → dot uses `statusNoCheck` token, aria-label="Status: No health check"
    - [ ] Root element has `role="status"`
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/components/StatusIndicator.jsx` (AC: #1–#5)
  - [ ] Props: `{ status }` (value from `STATUS` enum)
  - [ ] Token map:
    ```js
    const tokenMap = {
      [STATUS.ONLINE]: tokens.statusOnline,
      [STATUS.UNREACHABLE]: tokens.statusUnreachable,
      [STATUS.CHECKING]: tokens.statusChecking,
      [STATUS.NO_CHECK]: tokens.statusNoCheck,
    };
    ```
  - [ ] aria-label map:
    ```js
    const ariaMap = {
      [STATUS.ONLINE]: 'Status: Online',
      [STATUS.UNREACHABLE]: 'Status: Unreachable',
      [STATUS.CHECKING]: 'Status: Checking',
      [STATUS.NO_CHECK]: 'Status: No health check',
    };
    ```
  - [ ] Render: `<span role="status" aria-label={ariaMap[status]} style={{ color: tokenMap[status] }}>●  {displayLabel}</span>`
  - [ ] Display label: ONLINE → "ONLINE", UNREACHABLE → "UNREACHABLE", CHECKING → "CHECKING...", NO_CHECK → "NO CHECK"
- [ ] Task 3: Add `StatusIndicator` to `ServiceCard` (AC: #1)
  - [ ] Pass `status` prop from ServiceCard to StatusIndicator (Story 4.3 will wire the live value)
  - [ ] Default: `STATUS.CHECKING` for enabled cards, `STATUS.NO_CHECK` for disabled
- [ ] Task 4: Run all tests — green (AC: #6)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 3.1 — `ServiceCard` exists; Story 2.1 — theme tokens exist; Story 1.1 — `STATUS` enum exists.
**Token usage:** `useTheme()` in `StatusIndicator` provides `tokens`. Color dot = `style={{ color: tokenMap[status] }}`.
**aria-label exact strings (tested):**
- ONLINE → "Status: Online"
- UNREACHABLE → "Status: Unreachable"
- CHECKING → "Status: Checking"
- NO_CHECK → "Status: No health check"

Note the mixed case in aria-labels — these match the ACs exactly. Tests will verify exact strings.

### Project Structure Notes

New:
```
src/
├── components/
│   └── StatusIndicator.jsx
└── __tests__/components/
    └── StatusIndicator.test.jsx
```
Modified:
- `src/components/ServiceCard.jsx` (add StatusIndicator rendering)

### References

- FR7 (status indicators with aria-labels): `docs/terminus/portal/portal2/epics.md`
- TR3 (STATUS enum): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Health Polling"
- Story 4.2 (useHealthCheck hook — drives the status value)
- Story 7.2 (aria-labels audited in E2E)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
