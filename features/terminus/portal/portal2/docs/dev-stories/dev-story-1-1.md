# Story 1.1: Vite/React Project Scaffold with TDD Baseline

Status: done

## Story

As a developer,
I want a Vite/React project initialized with Vitest and React Testing Library,
so that every subsequent story has a runnable test suite before implementation begins.

## Acceptance Criteria

1. Running `npm install && npm test` passes all tests (baseline zero-file pass acceptable)
2. `npm run build` produces `dist/` with `index.html`
3. No hardcoded hex color values anywhere in `src/`
4. `src/config/services.js` exports an empty `SERVICES = []` placeholder
5. `src/config/metrics.js` exports an empty `METRICS = []` placeholder
6. `src/constants/status.js` exports `STATUS = { CHECKING, ONLINE, UNREACHABLE, NO_CHECK }`

## Tasks / Subtasks

- [ ] Task 1: Initialize Vite/React project (AC: #1, #2)
  - [ ] Replace existing `index.html` + any static assets with Vite/React scaffold
  - [ ] Run: `npm create vite@latest . -- --template react` (or manual init if non-interactive needed)
  - [ ] Verify `npm run build` produces `dist/index.html`
- [ ] Task 2: Install and configure Vitest + RTL (AC: #1)
  - [ ] `npm install -D vitest @vitest/coverage-v8 @testing-library/react @testing-library/jest-dom jsdom`
  - [ ] Create `vitest.config.js` with `environment: 'jsdom'` and `setupFiles: ['./src/test/setup.js']`
  - [ ] Create `src/test/setup.js` importing `@testing-library/jest-dom`
  - [ ] Add `"test": "vitest run"` and `"test:watch": "vitest"` to `package.json`
- [ ] Task 3: Write baseline test (TDD — write test first) (AC: #1)
  - [ ] Create `src/__tests__/baseline.test.js` — smoke test (e.g. `expect(1 + 1).toBe(2)`)
  - [ ] Run `npm test` — verify passes
- [ ] Task 4: Create config skeleton files (AC: #4, #5, #6)
  - [ ] `src/config/services.js` — `export const SERVICES = [];`
  - [ ] `src/config/metrics.js` — `export const METRICS = [];`
  - [ ] `src/constants/status.js` — `export const STATUS = Object.freeze({ CHECKING: 'CHECKING', ONLINE: 'ONLINE', UNREACHABLE: 'UNREACHABLE', NO_CHECK: 'NO_CHECK' });`
- [ ] Task 5: Audit for hardcoded hex values (AC: #3)
  - [ ] `grep -r '#[0-9a-fA-F]\{3,6\}' src/` must return empty (except comments)
  - [ ] Remove any hex values from Vite scaffold defaults

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Stack:** React 18+, Vite 5+, Vitest, @testing-library/react, jsdom
**TDD rule (NFR1):** Write the baseline test BEFORE wiring Vitest — verify it fails, then configure Vitest to make it pass.

### Project Structure Notes

Post-scaffold expected structure:
```
terminus-portal/
├── src/
│   ├── __tests__/baseline.test.js
│   ├── config/
│   │   ├── services.js
│   │   └── metrics.js
│   ├── constants/
│   │   └── status.js
│   ├── test/
│   │   └── setup.js
│   ├── App.jsx
│   └── main.jsx
├── dist/          (after build)
├── package.json
├── vite.config.js
└── vitest.config.js
```

The existing repo has a single `index.html` + nginx `Dockerfile`. Replace `index.html` with the Vite scaffold. Keep `Dockerfile` and `deploy/helm/` — those are addressed in Stories 1.2 and 1.3.

### References

- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Build & Tooling"
- NFR1 (TDD), NFR2 (static build): `docs/terminus/portal/portal2/epics.md`
- TR2 (config files), TR3 (STATUS enum): `docs/terminus/portal/portal2/epics.md`

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
