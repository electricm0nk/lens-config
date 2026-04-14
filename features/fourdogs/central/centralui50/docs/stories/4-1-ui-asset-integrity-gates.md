# Story 4-1: UI Asset Integrity Gates

## Goal
Prevent production deployment of broken or placeholder UI bundles.

## Acceptance Criteria

- Release workflows fail if required UI build artifacts are missing.
- Release workflows fail if placeholder-only index content is detected.
- Post-release smoke check validates styled authenticated shell.
