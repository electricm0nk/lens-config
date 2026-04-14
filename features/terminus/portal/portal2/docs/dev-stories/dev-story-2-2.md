# Story 2.2: localStorage Theme Persistence

Status: done

## Story

As a user,
I want my theme choice persisted across browser sessions,
so that I don't have to re-select my preferred theme on every visit.

## Acceptance Criteria

1. Terminal theme selected → close → reopen → Terminal theme is active
2. Invalid/unknown theme key in localStorage → default `spaceops` used
3. `localStorage.setItem` called with key `terminus-portal-theme` on every `toggleTheme()` call
4. Unit tests: fresh load (no key → default), valid key → correct theme, invalid key → default

## Tasks / Subtasks

- [ ] Task 1: Write failing tests FIRST (TDD) (AC: #3, #4)
  - [ ] `src/__tests__/theme/localStorage.test.jsx`:
    - [ ] Fresh load (no item in localStorage) → `themeName === 'spaceops'`
    - [ ] localStorage has `terminus-portal-theme: 'terminal'` → `themeName === 'terminal'`
    - [ ] localStorage has `terminus-portal-theme: 'invalid-value'` → `themeName === 'spaceops'`
    - [ ] `toggleTheme()` call → `localStorage.setItem` called with `('terminus-portal-theme', 'terminal')` or `'spaceops'`
  - [ ] Mock `localStorage` using `vi.stubGlobal` or vitest's built-in jsdom localStorage
  - [ ] Run tests — confirm FAIL (red)
- [ ] Task 2: Update `ThemeProvider` to read localStorage on init (AC: #1, #2)
  - [ ] In `ThemeProvider` initial state: read `localStorage.getItem('terminus-portal-theme')`
  - [ ] Validate: if value not in `['terminal', 'spaceops']`, fall back to `'spaceops'`
  - [ ] Set initial `themeName` from validated stored value
- [ ] Task 3: Update `toggleTheme()` to write localStorage (AC: #3)
  - [ ] After updating `themeName` state, call `localStorage.setItem('terminus-portal-theme', newTheme)`
- [ ] Task 4: Run all tests — green (AC: #4)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Depends on:** Story 2.1 — `ThemeProvider` must exist.
**localStorage key:** `terminus-portal-theme` (exact string — tested)
**Valid values:** `'terminal'` | `'spaceops'` — all others silently fall back to default.
**jsdom support:** `localStorage` is available in jsdom (Vitest default). Use `vi.stubGlobal` if needed, or `localStorage.clear()` in `beforeEach`.

### Project Structure Notes

Modify only:
```
src/
└── context/
    └── ThemeContext.jsx   (update ThemeProvider to add localStorage read/write)
```
New tests only:
```
src/
└── __tests__/
    └── theme/
        └── localStorage.test.jsx
```

### References

- FR5 (theme persistence): `docs/terminus/portal/portal2/epics.md`
- Story 2.1 (ThemeContext prerequisite)
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Theme System"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
