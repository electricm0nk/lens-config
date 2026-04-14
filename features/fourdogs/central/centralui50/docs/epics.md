# Epics: fourdogs-central-centralui50

## Epic 1: UI Foundation Recovery

Restore the intended visual system and eliminate placeholder or degraded rendering outcomes.

### Stories

- 1-1 Restore app shell style initialization and global token loading.
- 1-2 Replace fallback placeholder surfaces with designed empty states.
- 1-3 Add route smoke checks for post-login styled rendering.

## Epic 2: Theme and Interaction Recovery

Reinstate light/dark mode and core UX controls.

### Stories

- 2-1 Rewire theme provider and persistence.
- 2-2 Validate theme parity across dashboard and order detail views.
- 2-3 Add interaction regressions tests for key controls.

## Epic 3: Ordering and Kaylee Workflow Recovery

Restore the primary operational workflow used by operators.

### Stories

- 3-1 Restore Kaylee panel rendering and state lifecycle.
- 3-2 Restore order create/edit/submit interactions.
- 3-3 Add end-to-end smoke flow for create -> edit -> submit.

## Epic 4: Deployment Guardrails

Prevent broken UI assets from being deployed.

### Stories

- 4-1 Add UI bundle integrity assertions in release pipeline.
- 4-2 Enforce build-time checks against placeholder-only payloads.
- 4-3 Add post-deploy verification step for styled shell + key interaction entry points.
