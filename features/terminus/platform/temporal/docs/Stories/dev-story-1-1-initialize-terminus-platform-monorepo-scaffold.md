# Story 1.1: Initialize terminus.platform Monorepo Scaffold

Status: ready-for-dev

## Story

As a platform developer,
I want a fully scaffolded `terminus.platform` monorepo with npm workspaces, shared type definitions, and CI skeleton files,
so that all subsequent TypeScript implementation stories (worker, activities, Helm CI) have a stable, consistent foundation to build on.

## Acceptance Criteria

1. `package.json` with `workspaces: ["services/*", "shared/*"]` exists at repo root
2. Root `tsconfig.json` with `strict: true`; all service `tsconfig.json` files extend it
3. `shared/types/` package: `module.ts` (Module interface, ModulePayload<S> with required `schema`, RunContext), `index.ts`, `package.json`, `tsconfig.json`
4. `shared/lib/` package: `logger.ts` (pino factory with `service: terminus-platform-worker` default), `errors.ts` (ApplicationFailure helpers), `package.json`, `tsconfig.json`
5. `services/temporal/` directory scaffolded: `package.json`, `tsconfig.json`, `vitest.config.ts`, `src/` with empty `worker.ts`, empty `workflows/` and `activities/` dirs, `src/lib/` placeholder
6. `.github/workflows/ci.yml` — syntactically valid YAML skeleton; runs tests for changed services on PR
7. `.github/workflows/publish-images.yml` — syntactically valid YAML skeleton
8. `npm install` succeeds cleanly at repo root — zero errors
9. `npx tsc --noEmit` in `shared/types/` compiles clean — zero errors
10. Prometheus and Grafana stub directories (`services/grafana/`, `services/prometheus/`) are **NOT** created

## Tasks / Subtasks

- [ ] Task 0: Sync target repo to latest main and verify clean state (AC: all)
  - [ ] `cd TargetProjects/terminus/platform/terminus.platform && git checkout main && git pull origin main`
  - [ ] Confirm only `README.md` at repo root — stub state verified
  - [ ] **Constitutional override active (terminus Art. 5):** implementation work lands on `develop` in `terminus.platform` — no feature/epic/story branches required here

- [ ] Task 1: Write failing tests for `shared/types` before any implementation (AC: 3, 9) — RED phase (Art. 7/8)
  - [ ] Create `shared/types/src/module.test.ts` with failing tests covering:
    - `ModulePayload` requires `schema` field — test that a payload without `schema` fails type check or throws at runtime
    - `Module` interface has required `name`, `description`, `run` fields
    - `RunContext` has required `date`, `timezone`, `config` fields
    - Generic `ModulePayload<S>` — `schema` field is typed as `S`
  - [ ] Run tests — confirm they fail (compiletime or runtime)

- [ ] Task 2: Create root workspace skeleton (AC: 1, 2)
  - [ ] Create `package.json` at repo root:
    ```json
    {
      "name": "terminus-platform",
      "private": true,
      "workspaces": ["services/*", "shared/*"],
      "scripts": {
        "test": "npx vitest run",
        "typecheck": "tsc --noEmit"
      },
      "devDependencies": {
        "typescript": "^5.4.0",
        "vitest": "^1.6.0"
      }
    }
    ```
  - [ ] Create root `tsconfig.json`:
    ```json
    {
      "compilerOptions": {
        "strict": true,
        "target": "ES2022",
        "module": "NodeNext",
        "moduleResolution": "NodeNext",
        "esModuleInterop": true,
        "skipLibCheck": true,
        "declaration": true,
        "outDir": "dist"
      }
    }
    ```
  - [ ] Create `.gitignore` (node_modules, dist, coverage, *.js where appropriate)
  - [ ] Commit: `chore(scaffold): add root package.json and tsconfig`

- [ ] Task 3: Create `shared/types` package (AC: 3, 9) — GREEN phase (Art. 7/8)
  - [ ] Create `shared/types/package.json`:
    ```json
    {
      "name": "@terminus-platform/types",
      "version": "0.1.0",
      "main": "dist/index.js",
      "types": "dist/index.d.ts",
      "scripts": {
        "build": "tsc",
        "test": "vitest run",
        "typecheck": "tsc --noEmit"
      }
    }
    ```
  - [ ] Create `shared/types/tsconfig.json` extending root:
    ```json
    {
      "extends": "../../tsconfig.json",
      "compilerOptions": { "outDir": "dist", "rootDir": "src" },
      "include": ["src"]
    }
    ```
  - [ ] Create `shared/types/src/module.ts`:
    ```typescript
    export interface RunContext {
      date: string;       // ISO 8601
      timezone: string;   // IANA timezone e.g. "America/New_York"
      config: Record<string, unknown>;
    }

    export interface ModulePayload<S> {
      schema: S;          // required — no untyped payloads
      context: RunContext;
      data: unknown;
    }

    export interface Module {
      name: string;
      description: string;
      run(payload: ModulePayload<unknown>): Promise<void>;
    }
    ```
  - [ ] Create `shared/types/src/index.ts`: barrel export of all types
  - [ ] Run tests in `shared/types/` — confirm they now PASS (GREEN)
  - [ ] Run `npx tsc --noEmit` in `shared/types/` — confirm clean compile
  - [ ] Commit: `feat(shared/types): add Module, ModulePayload, RunContext interfaces`

- [ ] Task 4: Create `shared/lib` package (AC: 4)
  - [ ] Create `shared/lib/package.json` (similar to types, name `@terminus-platform/lib`)
  - [ ] Create `shared/lib/tsconfig.json` extending root
  - [ ] Create `shared/lib/src/logger.ts`:
    - pino factory function `createLogger({ module: string }): Logger`
    - Pre-sets log field `service: 'terminus-platform-worker'`
    - Returns configured pino logger with `module` field
  - [ ] Create `shared/lib/src/errors.ts`:
    - `nonRetryable(message: string, cause?: Error): ApplicationFailure` helper
    - `retryable(message: string, cause?: Error): ApplicationFailure` helper
    - Import from `@temporalio/activity` (declare as peer dependency, not installed yet)
  - [ ] Add `pino` as a dependency in `shared/lib/package.json`
  - [ ] Commit: `feat(shared/lib): add pino logger factory and ApplicationFailure helpers`

- [ ] Task 5: Create `services/temporal/` scaffold (AC: 5)
  - [ ] Create `services/temporal/package.json`:
    - name: `@terminus-platform/temporal-worker`
    - dependencies: `@terminus-platform/types` (workspace:*), `@terminus-platform/lib` (workspace:*)
    - devDependencies: `typescript`, `vitest`
    - Note `@temporalio/sdk` is NOT installed yet — this is scaffold only
  - [ ] Create `services/temporal/tsconfig.json` extending root
  - [ ] Create `services/temporal/vitest.config.ts`:
    ```typescript
    import { defineConfig } from 'vitest/config'
    export default defineConfig({
      test: {
        coverage: {
          thresholds: { lines: 80 },
          include: ['src/**']
        }
      }
    })
    ```
  - [ ] Create empty stubs: `src/worker.ts`, `src/workflows/.gitkeep`, `src/activities/.gitkeep`, `src/lib/.gitkeep`
  - [ ] Commit: `chore(services/temporal): scaffold package structure`

- [ ] Task 6: Create CI workflow skeletons (AC: 6, 7)
  - [ ] Create `.github/workflows/ci.yml`:
    ```yaml
    name: CI
    on:
      pull_request:
        branches: [main]
    jobs:
      test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - uses: actions/setup-node@v4
            with:
              node-version: '20'
          - run: npm install
          - run: npm test
    ```
  - [ ] Create `.github/workflows/publish-images.yml`:
    ```yaml
    name: Publish Images
    on:
      push:
        branches: [main]
    jobs:
      build:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          # Image build steps will be implemented in Story 3.3
    ```
  - [ ] Validate both files are syntactically valid YAML (`python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"`)
  - [ ] Commit: `ci: add CI and publish-images workflow skeletons`

- [ ] Task 7: Run full workspace verification (AC: 8, 9, 10)
  - [ ] `npm install` at repo root — confirm zero errors
  - [ ] `npx tsc --noEmit` in `shared/types/` — confirm clean
  - [ ] `npm test` — confirm `shared/types` tests pass (100% coverage on schema)
  - [ ] Confirm no `services/grafana/` or `services/prometheus/` directories exist
  - [ ] Confirm no `TargetProjects/` or BMAD tooling in the repo
  - [ ] Final commit if any fixups needed: `chore: workspace verification cleanups`
  - [ ] `git push origin develop`

## Dev Notes

### Constitutional Override — Develop-First Integration (CRITICAL)
**Terminus domain constitution Art. 5 is active for `terminus.platform`.** This means:
- **No** `feature/terminus-platform-temporal` branch
- **No** epic or story branches in `terminus.platform`
- **Implementation work lands on `develop`**
- Each task listed above lands on `develop`
- This is the authorized pattern — not a shortcut

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```
Current state: stub repo with `README.md` only.

### TDD Red-Green Requirement (Art. 7/8)
Task 1 MUST be executed before Task 3. Write the failing `module.test.ts` first (RED), then implement `module.ts` until tests pass (GREEN). This order is mandatory — do not skip or reverse.

100% line coverage is required on all `ModulePayload` schema validation in `shared/types`.

### Architecture Reference
Full directory tree: [docs/terminus/platform/temporal/architecture.md — Project Structure & Boundaries (Step 6)](docs/terminus/platform/temporal/architecture.md)

Key structure constraints:
- Tests co-located as `*.test.ts` alongside source — never in a separate `__tests__/` dir
- `shared/types` and `shared/lib` are the only cross-service shared code
- No service may import from another service
- `services/grafana/` and `services/prometheus/` are future initiatives — do NOT create them here

### Node.js / TypeScript Versions
- Node.js 20 (LTS) — consistent with CI workflow
- TypeScript ~5.4 — strict mode mandatory
- `module: NodeNext`, `moduleResolution: NodeNext` — required for ESM/CJS interop with Temporal SDK (Story 3.2)

### pino Logger Notes
`shared/lib/src/logger.ts` pre-sets `service: 'terminus-platform-worker'`. Each module additionally passes `{ module: 'module-name' }` when calling `createLogger()`. Required log fields per architecture: `timestamp`, `level`, `service`, `module`, `message`, `context`.

### ApplicationFailure in `shared/lib/src/errors.ts`
`@temporalio/activity` is a peer dependency — declare it in `shared/lib/package.json` but it will be installed in Story 3.2 when the full Temporal SDK is added. For now, the type import can be stubbed as:
```typescript
// errors.ts — stub for Story 1.1; full implementation in Story 3.2
export type ApplicationFailure = Error & { nonRetryable?: boolean }
export const nonRetryable = (msg: string, cause?: Error): ApplicationFailure => ...
```
Or import from a minimal local type until Temporal SDK is installed. Do not install `@temporalio/sdk` yet — that is Story 3.2's dependency.

### `ModulePayload.schema` Rule
Every `ModulePayload<S>` instance MUST have `schema` typed as `S`. No untyped payloads (`ModulePayload<unknown>` with no schema declaration) are permitted in production code. The `shared/types` tests must enforce this at the type level.

### Not In Scope
- Workflow or activity implementations (Story 3.2)
- Temporal SDK installation (Story 3.2)
- Helm chart (Story 3.1)
- k8s manifests (Stories 2.1, 2.2)
- DB SQL scripts (Story 1.2)
- CI image build logic (Story 3.3)
- Grafana / Prometheus directories

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — scaffold only, no external calls |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| TDD red-green | Org Art. 7 | ✅ NOTED — Task 1 (write tests) must precede Task 3 (implement) |
| BDD acceptance criteria | Org Art. 8 | ✅ PASS — all ACs have corresponding test task |
| Security first | Org Art. 9 | ✅ PASS — no secrets, no credentials in this story |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
