# Story 3.2: TypeScript Worker Scaffold — DailyBriefingWorkflow

Status: done

## Story

As a platform developer,
I want a TypeScript Temporal worker that connects to the Temporal server, registers `DailyBriefingWorkflow` and `briefingActivities`, and polls the `terminus-platform` task queue,
so that the platform has a working worker skeleton on which the `terminus-agent-dailybriefing` initiative can build full activity implementations.

## Acceptance Criteria

1. `services/temporal/src/worker.ts`: connects to `temporal-frontend.temporal.svc.cluster.local:7233`, registers `DailyBriefingWorkflow` and `briefingActivities`, polls `terminus-platform` task queue
2. `services/temporal/src/workflows/dailyBriefingWorkflow.ts`: skeleton workflow function — `async function dailyBriefingWorkflow(payload: ModulePayload<DailyBriefingSchema>): Promise<void>` — calls at least one activity
3. `services/temporal/src/workflows/dailyBriefingWorkflow.test.ts`: co-located test; mocks activity calls; TDD scaffolded before implementation
4. `services/temporal/src/activities/briefingActivities.ts`: one placeholder activity (`fetchBriefingData`) — returns stub `ModulePayload<BriefingDataSchema>` with `schema` declared
5. `services/temporal/src/activities/briefingActivities.test.ts`: co-located test, 80%+ line coverage
6. `services/temporal/src/lib/logger.ts` and `src/lib/errors.ts`: pino logger factory + ApplicationFailure helpers
7. All workflow functions use `@temporalio/sdk` — no `Date.now()`, `Math.random()`, or non-deterministic APIs in workflow code
8. All activity errors re-thrown as `ApplicationFailure.nonRetryable()` or `ApplicationFailure` (never raw `Error`)
9. `npm run test` passes with 80%+ line coverage and 100% on `ModulePayload` schema validation
10. **N1 (health probe):** `services/temporal/src/lib/health.ts` — gRPC probe script attempting connection to `temporal-frontend.temporal.svc.cluster.local:7233`, exits 0/1. Referenced in Helm chart `livenessProbe`.
11. `services/temporal/package.json` declares workspace dependency on `shared/types` and `shared/lib`

## Tasks / Subtasks

- [ ] Task 1: Write failing tests for workflow and activities (AC: 3, 5, 9) — RED phase (Art. 7/8)
  - [ ] Create `services/temporal/src/workflows/dailyBriefingWorkflow.test.ts`:
    - Mock `briefingActivities.fetchBriefingData`
    - Test that workflow calls `fetchBriefingData` activity
    - Test that workflow accepts `ModulePayload<DailyBriefingSchema>` input
  - [ ] Create `services/temporal/src/activities/briefingActivities.test.ts`:
    - Test that `fetchBriefingData` returns a `ModulePayload<BriefingDataSchema>` with `schema` declared
    - Test that errors are thrown as `ApplicationFailure`, not raw `Error`
  - [ ] Run tests — confirm they fail (RED)

- [ ] Task 2: Install Temporal SDK dependencies (AC: 1, 11)
  - [ ] Add to `services/temporal/package.json`:
    - `@temporalio/worker`, `@temporalio/workflow`, `@temporalio/activity`, `@temporalio/client`
    - `@terminus-platform/types: "workspace:*"`
    - `@terminus-platform/lib: "workspace:*"`
  - [ ] `npm install` at repo root — confirm clean
  - [ ] Commit: `chore(temporal-worker): add Temporal SDK dependencies`

- [ ] Task 3: Implement activity stubs (AC: 4, 5, 8) — GREEN phase
  - [ ] Implement `services/temporal/src/activities/briefingActivities.ts`:
    - Export `fetchBriefingData(payload: ModulePayload<DailyBriefingSchema>): Promise<ModulePayload<BriefingDataSchema>>`
    - Return stub response with `schema: BriefingDataSchema` declared
    - All errors re-thrown as `ApplicationFailure.nonRetryable(message, cause)`
  - [ ] Define `DailyBriefingSchema` and `BriefingDataSchema` types (stub — full schema in `terminus-agent-dailybriefing`)
  - [ ] Run activity tests — confirm they pass (GREEN)
  - [ ] Commit: `feat(temporal-worker): add briefingActivities stub`

- [ ] Task 4: Implement workflow skeleton (AC: 2, 3, 7) — GREEN phase
  - [ ] Implement `services/temporal/src/workflows/dailyBriefingWorkflow.ts`:
    - `export async function DailyBriefingWorkflow(payload: ModulePayload<DailyBriefingSchema>): Promise<void>`
    - Uses `proxyActivities<typeof briefingActivities>()` — no direct imports from activities
    - Calls `fetchBriefingData(payload)` activity
    - No `Date.now()`, `Math.random()`, or non-deterministic APIs
  - [ ] Note: workflow function name must be `DailyBriefingWorkflow` (PascalCase — matches Temporal workflow type registration)
  - [ ] Run workflow tests — confirm they pass (GREEN)
  - [ ] Commit: `feat(temporal-worker): add DailyBriefingWorkflow skeleton`

- [ ] Task 5: Implement worker entrypoint (AC: 1)
  - [ ] Implement `services/temporal/src/worker.ts`:
    - Connect to `temporal-frontend.temporal.svc.cluster.local:7233`
    - Register `DailyBriefingWorkflow` workflow type
    - Register `briefingActivities` activities
    - Poll task queue `terminus-platform`
    - Namespace: `default`
  - [ ] Commit: `feat(temporal-worker): add worker entrypoint`

- [ ] Task 6: Implement logger and errors helpers (AC: 6)
  - [ ] Create `services/temporal/src/lib/logger.ts`: re-export or extend `shared/lib` pino factory with `module: 'temporal-worker'`
  - [ ] Create `services/temporal/src/lib/errors.ts`: re-export `ApplicationFailure` helpers from `shared/lib`
  - [ ] Commit: `feat(temporal-worker): add logger and errors lib helpers`

- [ ] Task 7: Implement health probe (AC: 10)
  - [ ] Create `services/temporal/src/lib/health.ts`:
    - Attempt gRPC connection to `temporal-frontend.temporal.svc.cluster.local:7233`
    - Exit 0 on success, exit 1 on failure (within timeout)
  - [ ] Update Helm chart `livenessProbe` in Story 3.1 template to reference this script
  - [ ] Commit: `feat(temporal-worker): add health probe script`

- [ ] Task 8: Final verification (AC: 9)
  - [ ] `npm run test` at repo root — confirm 80%+ line coverage
  - [ ] `npx tsc --noEmit` in `services/temporal/` — confirm clean compile
  - [ ] `git push origin develop`

## Dev Notes

### Target Repo Path
```
TargetProjects/terminus/platform/terminus.platform/
```

### Critical: Workflow Function Name
The Temporal workflow type is registered by function name. The implementation **must** use `DailyBriefingWorkflow` (PascalCase) — not `dailyBriefingWorkflow`. This was a defect found and fixed during smoke testing (Story 4.1): the original camelCase name caused a `WorkflowNotFound` error.

### TDD Red-Green Requirement (Art. 7/8)
Task 1 (write failing tests) MUST precede Tasks 3 and 4 (implementations). Do not skip or reverse.

### Temporal Determinism Rule
Workflow functions (`DailyBriefingWorkflow`) MUST NOT call:
- `Date.now()` or `new Date()` — use `workflow.sleep()` or Temporal timers
- `Math.random()` — use deterministic alternatives
- Any I/O, network, or filesystem operations

All non-deterministic operations belong exclusively in activities.

### ModulePayload.schema Rule
Every `ModulePayload<S>` instance MUST have `schema` typed as `S`. This is enforced by `shared/types` — no untyped payloads.

### Activity Error Wrapping
ALL activity errors must be wrapped in `ApplicationFailure`:
```typescript
try {
  // operation
} catch (err) {
  throw ApplicationFailure.nonRetryable(`fetchBriefingData failed: ${err.message}`, err as Error)
}
```
Never `throw err` (raw Error) from an activity.

### Constitutional Override — Develop-First Integration
Implementation work lands on `develop` in `terminus.platform` (Terminus Art. 5 override active).

### Not In Scope
- Full `DailyBriefingWorkflow` activity implementations (terminus-agent-dailybriefing initiative)
- CI image build (Story 3.3)
- `DailyBriefingSchema` full definition — stubs only; `terminus-agent-dailybriefing` owns the schema

## Pre-Implementation Constitution Gates

| Gate | Article | Result |
|------|---------|--------|
| Track declared | Org Art. 1 | ✅ PASS — `tech-change` |
| Architecture documented | Org Art. 3 | ✅ PASS — `docs/terminus/platform/temporal/architecture.md` |
| No confidential data | Org Art. 4 | ✅ PASS — no credentials in source code |
| Git discipline | Org Art. 5 | ✅ PASS — develop-first integration authorized by terminus Art. 5 |
| TDD red-green | Org Art. 7 | ✅ NOTED — Task 1 (tests) must precede Tasks 3/4 (implementations) |
| BDD acceptance criteria | Org Art. 8 | ✅ PASS — all ACs have corresponding test task |
| Security first | Org Art. 9 | ✅ PASS — no secrets in source; ApplicationFailure wrapping prevents leak |
| Repo as source of truth | Org Art. 10 | ✅ PASS — all context from committed docs |
| Develop-first integration | Terminus Art. 5 | ✅ OVERRIDE ACTIVE — land implementation work on `develop` in `terminus.platform` |

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (copilot)

### Debug Log References

### Completion Notes List

### File List
