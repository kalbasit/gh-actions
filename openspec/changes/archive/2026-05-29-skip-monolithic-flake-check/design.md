# Design: skip-monolithic-flake-check

## Context

`build.yml` (oci) runs `nix flake check` + `Build coverage` + Codecov inline per
matrix system; `ci.yml` also has a standalone `flake-check` matrix job for the
`oci: false` path. A consumer fanning checks out into its own matrix needs to
suppress both, without affecting other consumers.

## Goals / Non-Goals

**Goals**
- One opt-in boolean that disables the inline check/coverage everywhere.
- Default-true: zero change for existing consumers.

**Non-Goals**
- The consumer-side fan-out matrix; OCI build/manifest changes.

## Decisions

### D1 — `run_flake_check` boolean, default true
Added to both `build.yml` and `ci.yml`. In `build.yml` it ANDs into the existing
`if:` of the **Flake check**, **Build coverage**, and **Upload coverage to
Codecov** steps. In `ci.yml` it ANDs into the standalone `flake-check` job's
`if:` (`oci == false && run_flake_check`) and is forwarded to `build.yml`. The
OCI image build/push and `oci-manifest` are untouched, so a consumer with
`run_flake_check: false` still gets images + multi-arch manifest.

*Alternative:* a generic fan-out matrix inside the reusable workflow — larger,
deferred; this minimal switch lets the consumer own the matrix.

## Risks / Trade-offs

- **Consumer sets `run_flake_check: false` but forgets its own matrix** → checks
  don't run at all. Consumer responsibility; documented in the input.
- **Coverage**: with `false`, the workflow uploads no coverage — the consumer
  must run its own coverage job. Documented.

## Migration Plan

1. Land this input (default true → inert for all current consumers).
2. ncps sets `run_flake_check: false` and adds its fan-out + coverage jobs.
3. **Rollback:** consumer unsets the input.

## Open Questions
- None.
