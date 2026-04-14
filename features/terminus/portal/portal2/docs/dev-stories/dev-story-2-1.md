# Story 2.1: Theme Token Objects and Context Provider

Status: done

## Story

As a developer,
I want `ThemeContext` with Terminal and Space Ops token objects and a `useTheme()` hook,
so that every component can access typed tokens without prop drilling or hardcoded values.

## Acceptance Criteria

1. `ThemeProvider` wraps the application at root; `useTheme()` returns `{ tokens, themeName, toggleTheme }`
2. `tokens` includes all required keys: `bg`, `bgSurface`, `bgCard`, `text`, `textMuted`, `accent`, `accentDim`, `border`, `borderActive`, `statusOnline`, `statusUnreachable`, `statusChecking`, `statusNoCheck`, `fontFamily`, `scanlines`
3. No component file contains a hardcoded hex color value
4. Unit tests verify `terminal` and `spaceops` token objects contain all required keys
5. Unit tests verify `useTheme()` throws outside of `ThemeProvider`

## Tasks / Subtasks

- [ ] Task 1: Write failing tests FIRST (TDD) (AC: #4, #5)
  - [ ] `src/__tests__/theme/tokens.test.js`:
    - [ ] `terminal` token object has all 15 required keys
    - [ ] `spaceops` token object has all 15 required keys
  - [ ] `src/__tests__/theme/ThemeContext.test.jsx`:
    - [ ] `useTheme()` throws when called outside `ThemeProvider`
    - [ ] `useTheme()` returns `{ tokens, themeName, toggleTheme }` inside provider
    - [ ] Default theme is `spaceops`
  - [ ] Run tests — confirm they FAIL (red)
- [ ] Task 2: Create Terminal theme token object (AC: #2)
  - [ ] `src/themes/terminal.js` — dark green/CRT palette, `scanlines: true`
    - bg: '#0a0f0a', accent: '#00ff41', fontFamily: "'Courier New', monospace", etc.
- [ ] Task 3: Create Space Ops theme token object (AC: #2)
  - [ ] `src/themes/spaceops.js` — navy/HUD palette, `scanlines: false`
    - bg: '#0d1b2a', accent: '#4cc9f0', fontFamily: "'Inter', sans-serif", etc.
- [ ] Task 4: Create ThemeContext.jsx (AC: #1)
  - [ ] `src/context/ThemeContext.jsx` — `createContext`, `ThemeProvider`, `useTheme` hook
  - [ ] `ThemeProvider` state: `themeName` (default `'spaceops'`), computed `tokens`
  - [ ] `useTheme()` guard: throws if context is null (outside provider)
- [ ] Task 5: Wrap App root in `ThemeProvider` (AC: #1)
  - [ ] `src/main.jsx`: `<ThemeProvider><App /></ThemeProvider>`
- [ ] Task 6: Verify no hardcoded hex values (AC: #3)
  - [ ] `grep -rn '#[0-9a-fA-F]\{3,6\}' src/ --include="*.jsx" --include="*.js"` — only theme files allowed
- [ ] Task 7: Run all tests — green (AC: #4, #5)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Key design constraint (TR1):** Tokens are plain JS objects — NO CSS variables, NO CSS-in-JS library. Components use inline styles or pass token values to style props.
**scanlines token:** `boolean` — `true` for Terminal, `false` for Space Ops. Used in Story 2.3 to conditionally render CRT overlay.

Required token keys (both themes must have all of these):
```
bg, bgSurface, bgCard, text, textMuted, accent, accentDim,
border, borderActive,
statusOnline, statusUnreachable, statusChecking, statusNoCheck,
fontFamily, scanlines
```

Terminal palette suggestion: bg='#0a0f0a', text='#00ff41', accent='#00ff41', statusOnline='#00ff41', statusUnreachable='#ff3333', statusChecking='#ffaa00', fontFamily="'Courier New', monospace", scanlines=true

Space Ops palette suggestion: bg='#0d1b2a', text='#e0f0ff', accent='#4cc9f0', statusOnline='#4ade80', statusUnreachable='#f87171', statusChecking='#fbbf24', fontFamily="'Inter', sans-serif", scanlines=false

### Project Structure Notes

New files:
```
src/
├── themes/
│   ├── terminal.js
│   └── spaceops.js
├── context/
│   └── ThemeContext.jsx
└── __tests__/
    └── theme/
        ├── tokens.test.js
        └── ThemeContext.test.jsx
```

### References

- TR1 (ThemeContext, useTheme, token objects): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Theme System"
- Story 2.2 (localStorage persistence — extends ThemeProvider)
- Story 2.3 (scanline overlay uses `tokens.scanlines`)

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
