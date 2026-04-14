# Story 5.1: Header Component with Cluster Identity

Status: done

## Story

As a user,
I want a header displaying the platform name and cluster details,
so that I always know which environment the portal is connected to.

## Acceptance Criteria

1. Header displays: platform name ("Terminus Platform Portal"), cluster name ("trantor"), domain ("hintzmann.net"), technology ("k3s")
2. All text colors use theme tokens — no hardcoded hex values
3. Header rendered as `<header>` element with `role="banner"`
4. Component tests verify: correct text rendered, all styles reference tokens

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #4)
  - [ ] `src/__tests__/components/Header.test.jsx`:
    - [ ] Renders text "Terminus Platform Portal"
    - [ ] Renders "trantor"
    - [ ] Renders "hintzmann.net"
    - [ ] Renders "k3s"
    - [ ] Root element is `<header>` with `role="banner"`
    - [ ] No inline style contains hardcoded hex value (inspect style objects)
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `src/components/Header.jsx` (AC: #1, #2, #3)
  - [ ] Root: `<header role="banner" style={{ background: tokens.bgSurface, ...}}>``
  - [ ] Left side: platform title "TERMINUS PLATFORM PORTAL" in `tokens.accent`
  - [ ] Sub-line: cluster details — "trantor · hintzmann.net · k3s" in `tokens.textMuted`
  - [ ] Right side: placeholder for controls (ThemeToggle + RefreshButton added in Story 5.3)
  - [ ] All colors from `useTheme()` tokens
- [ ] Task 3: Render Header in `App.jsx` at the top (AC: #1)
  - [ ] Replace any existing title/header markup
- [ ] Task 4: Run all tests — green (AC: #4)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 2.1 — `useTheme()` must exist for token access.
**Static values (hardcoded in JSX — these are identity strings, not colors):**
- Platform name: "Terminus Platform Portal" (or "TERMINUS PLATFORM PORTAL" for terminal aesthetic)
- Cluster: "trantor"
- Domain: "hintzmann.net"
- Technology: "k3s"

**Future slots:**
- Story 5.2 adds `onlineCount`, `totalCount`, `lastChecked` props → summary display
- Story 5.3 moves ThemeToggle + RefreshButton into the right side of Header

**Suggested layout:**
```
┌─────────────────────────────────────────────────────┐
│ TERMINUS PLATFORM PORTAL         [controls area]    │
│ trantor · hintzmann.net · k3s    [placeholder]      │
└─────────────────────────────────────────────────────┘
```

### Project Structure Notes

New:
```
src/
├── components/
│   └── Header.jsx
└── __tests__/components/
    └── Header.test.jsx
```
Modified:
- `src/App.jsx` (add `<Header />` at top)

### References

- FR3 (header: cluster identity): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Header"
- Story 5.2 (adds health summary props)
- Story 5.3 (adds ThemeToggle + RefreshButton into Header)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
