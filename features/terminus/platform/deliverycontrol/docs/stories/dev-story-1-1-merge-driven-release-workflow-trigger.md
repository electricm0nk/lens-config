# Story 1.1: Merge-Driven Release Workflow Trigger

Status: ready-for-dev

## Story

As a service developer,
I want merged PRs to `develop` and `main` to trigger the release workflow automatically,
so that release approval is expressed only through git merge and not through a separate operator action.

## Acceptance Criteria

1. The service repository contains `.github/workflows/release.yml` wired to `pull_request` `closed` events for `develop` and `main`
2. The workflow executes only when `github.event.pull_request.merged == true`
3. The workflow ignores unmerged PR closures and non-release target branches
4. The workflow resolves `develop -> dev` and `main -> prod` as the canonical environment mapping for downstream jobs
5. The workflow exposes resolved environment and service identity as outputs for downstream jobs
6. The workflow summary and inline documentation state that PR merge is the only normal release approval signal
7. The workflow does not instruct operators to use Semaphore for normal releases

## Tasks / Subtasks

- [ ] Create `.github/workflows/release.yml` in the target service repository (AC: 1)
- [ ] Configure the workflow trigger for `pull_request` `closed` on `develop` and `main` only (AC: 1, 3)
- [ ] Add an explicit merged-PR guard using `github.event.pull_request.merged == true` (AC: 2)
- [ ] Implement a branch-to-environment resolution step that maps `develop` to `dev` and `main` to `prod` (AC: 4)
  - [ ] Emit the resolved environment as a job output for downstream jobs (AC: 5)
  - [ ] Emit the resolved service identity or repository-derived service name as a job output for downstream jobs (AC: 5)
- [ ] Add workflow summary text that records the target branch, resolved environment, and approval model (AC: 6)
- [ ] Verify the workflow text and comments describe Semaphore as break-glass only, never as the normal path (AC: 7)
- [ ] Validate the workflow against three scenarios: merged to `develop`, merged to `main`, closed without merge (AC: 2, 3, 4)

## Dev Notes

### Core Release Contract

This story establishes the entry point of the entire deliverycontrol initiative. The new release contract is explicit:

> A PR merge is the release approval. No further human action is required or permitted after merge.

That means this workflow must behave as the normal entry path and must not preserve old Semaphore-first assumptions in comments, docs, or summary output.

### Workflow Shape

The workflow should be based on this structure:

```yaml
name: Release
on:
  pull_request:
    types: [closed]
    branches: [develop, main]

jobs:
  build-and-push:
    if: github.event.pull_request.merged == true
```

Story 1.1 does not need to implement the full build/push or promotion logic yet. It must establish the correct trigger, gating, and output contract that later stories depend on.

### Canonical Mapping

The approved environment contract is fixed:

- `develop` -> `dev`
- `main` -> `prod`

This mapping is reused later by manifest promotion, Temporal workflow invocation, ArgoCD app naming, namespaces, and smoke-test template routing. Do not invent alternate branch aliases or environment names in this story.

### Scope Boundary

This story is about the workflow trigger and release entry semantics only. Do not pull in later-sprint concerns prematurely:

- Do not implement Docker build/push in this story
- Do not implement manifest promotion in this story
- Do not implement `temporal workflow start` in this story
- Do not add direct `kubectl apply` or `helm install` logic anywhere in the workflow

Those belong to later stories and are intentionally sequenced after this foundation.

### Required Output Contract for Later Stories

Downstream jobs will need at least:

- resolved environment (`dev` or `prod`)
- service identity / repository-derived service name
- branch context for audit output

Use explicit job outputs so later jobs do not need to re-derive the same information inconsistently.

### Audit and Operator Messaging

The workflow summary should make the release decision legible in GitHub Actions:

- target branch
- resolved environment
- merge commit / PR context
- statement that the merge itself is the approval

Operator-facing text must reinforce:

- GitHub Actions is the primary release entry path
- Semaphore is break-glass only

### Project Structure Notes

This initiative spans multiple implementation surfaces, but Story 1.1 is expected to start in a service repository:

- service repo: `.github/workflows/release.yml`
- supporting contracts documented in the control repo

Do not modify `terminus.infra` or `terminus.platform` in this story unless a tiny compatibility note or comment is strictly necessary. Those repos are addressed in later stories.

### Testing Requirements

Validate the workflow logic with scenario-based review at minimum:

1. PR merged into `develop` -> workflow runs, resolves `dev`
2. PR merged into `main` -> workflow runs, resolves `prod`
3. PR closed without merge -> workflow does not proceed

If the target repo already has CI for workflow linting or local workflow validation, use it. If not, at least ensure expression syntax and outputs are internally consistent.

### References

- [docs/terminus/platform/deliverycontrol/epics.md](docs/terminus/platform/deliverycontrol/epics.md#L74)
- [docs/terminus/platform/deliverycontrol/architecture.md](docs/terminus/platform/deliverycontrol/architecture.md#L140)
- [docs/cicd.md](docs/cicd.md#L31)
- [docs/terminus/architecture.md](docs/terminus/architecture.md#L254)

## Dev Agent Record

### Agent Model Used

_to be filled at implementation_

### Debug Log References

### Completion Notes List

### File List