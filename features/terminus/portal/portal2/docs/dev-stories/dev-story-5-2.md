# Story 5.2: Live Health Summary and Last-Checked Timestamp

Status: done

## Story

As a user,
I want the header to show how many services are online and when the last check ran,
so that I have an at-a-glance view of platform health without scanning every card.

## Acceptance Criteria

1. Header displays "X/Y ONLINE" where X = `STATUS.ONLINE` count, Y = `healthCheck.enabled: true` count
2. If any services are `STATUS.UNREACHABLE`, badge "N UNREACHABLE" shown in `statusUnreachable` token color
3. "Last checked: HH:MM:SS" shown using most recent completed poll timestamp
4. During initial load (all enabled services still CHECKING), displays "Checking..." instead of X/Y
5. Unit tests: all online, some unreachable, all checking initial state

## Tasks / Subtasks

- [ ] Task 1: Write failing unit tests FIRST (TDD) (AC: #5)
  - [ ] `src/__tests__/components/Header.summary.test.jsx`:
    - [ ] All enabled online: shows "8/8 ONLINE", no unreachable badge
    - [ ] 6 online, 2 unreachable: shows "6/8 ONLINE" + "2 UNREACHABLE" badge
    - [ ] All checking (initial load): shows "Checking..."
    - [ ] `lastChecked` is a Date → shows "Last checked: HH:MM:SS"
    - [ ] `lastChecked` is null → hides last-checked line
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Update `Header.jsx` to accept summary props (AC: #1, #2, #3, #4)
  - [ ] Props: `{ onlineCount, totalCount, unreachableCount, lastChecked, isPolling }`
  - [ ] Summary display:
    ```jsx
    {isPolling && onlineCount === 0 ? (
      <span style={{ color: tokens.textMuted }}>Checking...</span>
    ) : (
      <span style={{ color: tokens.accent }}>{onlineCount}/{totalCount} ONLINE</span>
    )}
    {unreachableCount > 0 && (
      <span style={{ color: tokens.statusUnreachable }}>{unreachableCount} UNREACHABLE</span>
    )}
    {lastChecked && (
      <span style={{ color: tokens.textMuted }}>
        Last checked: {lastChecked.toLocaleTimeString()}
      </span>
    )}
    ```
- [ ] Task 3: Compute summary in `App.jsx` and pass to Header (AC: #1, #2)
  - [ ] `const totalCount = SERVICES.filter(s => s.healthCheck.enabled).length`
  - [ ] `const onlineCount = Object.values(statusMap).filter(s => s === STATUS.ONLINE).length`
  - [ ] `const unreachableCount = Object.values(statusMap).filter(s => s === STATUS.UNREACHABLE).length`
  - [ ] Pass all to `<Header onlineCount={onlineCount} totalCount={totalCount} unreachableCount={unreachableCount} lastChecked={lastChecked} isPolling={isPolling} />`
- [ ] Task 4: Run all tests — green (AC: #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 4.3 — `statusMap`, `isPolling`, `lastChecked` state exist in App; Story 5.1 — `Header` component exists.
**"Checking..." display logic:** Show "Checking..." when poll is in progress AND no services have resolved yet (`onlineCount === 0` is a proxy). Once at least one result has resolved, switch to "X/Y ONLINE" immediately.
**Total count (Y):** Only count services with `healthCheck.enabled: true` — currently 8 (all except fourdogs). This is the denominator for the summary.
**lastChecked formatting:** Use `Date.toLocaleTimeString()` for HH:MM:SS in the user's locale. No external library needed.
**SERVICES enabled count check:** `SERVICES.filter(s => s.healthCheck.enabled).length` — computed once (static), can be a constant.

### Project Structure Notes

Modified:
- `src/components/Header.jsx` (add summary props + display)
- `src/App.jsx` (compute onlineCount, unreachableCount, pass to Header)

New tests:
```
src/
└── __tests__/components/
    └── Header.summary.test.jsx
```

### References

- FR3 (header: X/Y ONLINE, last-checked timestamp): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Header"
- Story 4.3 (statusMap, lastChecked state source)
- Story 5.1 (Header component prerequisite)
- Story 5.3 (controls added to Header)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
