# Story 2.3: Theme Toggle and CRT Scanline Overlay

Status: done

## Story

As a user,
I want to toggle between Terminal and Space Ops themes with a button,
so that I can switch between the dark-green CRT aesthetic and the HUD aesthetic.

## Acceptance Criteria

1. Clicking theme toggle button immediately updates all themed elements without page reload
2. When active theme is Terminal (`tokens.scanlines === true`), a fixed-position scanline overlay div is rendered
3. When active theme is Space Ops (`tokens.scanlines === false`), the scanline overlay is NOT rendered
4. `aria-label` on toggle button: "Switch to Terminal theme" or "Switch to Space Ops theme" based on current theme
5. Component tests verify overlay renders/disappears on theme toggle

## Tasks / Subtasks

- [ ] Task 1: Write failing component tests FIRST (TDD) (AC: #4, #5)
  - [ ] `src/__tests__/theme/ThemeToggle.test.jsx`:
    - [ ] Toggle button renders with correct `aria-label` ("Switch to Terminal theme" when spaceops active)
    - [ ] Clicking toggle calls `toggleTheme()`
    - [ ] When `tokens.scanlines === true`, scanline overlay div is in DOM
    - [ ] When `tokens.scanlines === false`, scanline overlay div is NOT in DOM
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Create `ThemeToggle` button component (AC: #1, #4)
  - [ ] `src/components/ThemeToggle.jsx`
  - [ ] Calls `toggleTheme()` from `useTheme()` on click
  - [ ] `aria-label`: when current theme is `spaceops` → "Switch to Terminal theme"; when `terminal` → "Switch to Space Ops theme"
  - [ ] All styles (button bg, text, border) use theme tokens — no hardcoded hex
- [ ] Task 3: Create scanline overlay (AC: #2, #3)
  - [ ] In `App.jsx` (or root layout): conditionally render `<div className="scanlines-overlay" />` when `tokens.scanlines === true`
  - [ ] Overlay styles: `position: fixed`, `top: 0`, `left: 0`, `width: '100%'`, `height: '100%'`, `pointerEvents: 'none'`, `zIndex: 9999`
  - [ ] Scanline pattern: CSS repeating-linear-gradient (inline style with token value, or fixed pattern)
  - [ ] When `tokens.scanlines === false` → overlay not rendered
- [ ] Task 4: Run all tests — green (AC: #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 2.2 — `ThemeProvider` with localStorage and `toggleTheme()` must exist.
**Note:** `ThemeToggle` is a standalone component here. It will be relocated into `Header` in Story 5.3. For now, render it in `App.jsx` near the top.
**aria-label logic:** The label describes the action ("Switch TO Terminal") not the current state. When user is on `spaceops`, the button switches TO `terminal`.

Scanline overlay implementation:
```jsx
{tokens.scanlines && (
  <div
    aria-hidden="true"
    style={{
      position: 'fixed', top: 0, left: 0, width: '100%', height: '100%',
      pointerEvents: 'none', zIndex: 9999,
      background: 'repeating-linear-gradient(transparent 0px, transparent 2px, rgba(0,0,0,0.05) 2px, rgba(0,0,0,0.05) 4px)'
    }}
  />
)}
```

### Project Structure Notes

New files:
```
src/
└── components/
    └── ThemeToggle.jsx
```
New tests:
```
src/
└── __tests__/theme/
    └── ThemeToggle.test.jsx
```
Modified:
```
src/App.jsx   (add ThemeToggle + scanline overlay)
```

### References

- FR5 (theme toggle): `docs/terminus/portal/portal2/epics.md`
- TR1 (no hardcoded hex): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Theme System"
- Story 5.3 (ThemeToggle moves into Header)
- Story 7.2 (aria-label audited)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
