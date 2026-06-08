## Why

The reusable `ci.yml` `coverage` job runs concurrently with the check-producing
jobs (`flake-check` / `check-matrix` / `checks` / `build`) instead of after them.
Coverage derivations typically depend on the very test/cohort derivations those
jobs build and push to Cachix, so running in parallel means the coverage job
wins zero cache hits and rebuilds the entire instrumented test suite from
scratch. In the `ncps` consumer this regressed the coverage job from ~33s to
~6m20s — the job now serially re-runs the whole test matrix a second time. The
regression landed when `ncps` PR #1363 migrated coverage off its bespoke job
(which had `needs: checks`) onto this shared job, which dropped that dependency.

## What Changes

- The `coverage` job in `.github/workflows/ci.yml` SHALL declare a `needs`
  dependency on the check-producing jobs (`check-mode-guard`, `flake-check`,
  `check-matrix`, `checks`, `build`) so it only starts after those jobs have
  populated Cachix, restoring binary-cache reuse for coverage's dependencies.
- The job's `if:` condition SHALL be rewritten to tolerate the
  mode-dependent skips among its new dependencies (e.g. `flake-check` is skipped
  in `matrix` mode; `build` is skipped when `oci: false`) — a skipped `needs`
  job would otherwise cascade-skip coverage. Gating moves to a
  `!cancelled()`-style guard that still honours the existing `inputs.coverage`
  toggle and the fork-PR / missing-token skip.
- No new inputs, secrets, or consumer-side changes. The behaviour fix is wholly
  inside this repo's `ci.yml`; consumers tracking `@main` pick it up for free.

## Capabilities

### New Capabilities
<!-- none -->

### Modified Capabilities
- `reusable-ci`: The "Coverage upload" requirement gains an ordering guarantee —
  the `coverage` job runs after the check-producing jobs so it reuses their
  Cachix-pushed build artifacts rather than rebuilding them, and its gating is
  specified to survive mode-dependent dependency skips.

## Impact

- `.github/workflows/ci.yml` — `coverage` job `needs` + `if` only. No change to
  `build.yml`, `generate.yml`, or any input/secret contract.
- Consumers: `ncps` (coverage returns to ~30s and stops double-running the
  suite); `swm` / `signal-api-receiver` unaffected unless they enable coverage.
- CI critical path for coverage-enabled consumers drops from
  `max(checks, full-rebuild)` to `checks + merge`, and wasted duplicate compute
  is eliminated.
