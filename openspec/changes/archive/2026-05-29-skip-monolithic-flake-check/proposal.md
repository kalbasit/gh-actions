# Proposal: skip-monolithic-flake-check

## Why

The reusable workflow runs `nix flake check` (and coverage) inline — in the
standalone `flake-check` job and inside the OCI `build` job. A consumer that
wants to fan the checks out into its own parallel matrix (one job per
`nix build .#checks.<system>.<name>`, for much faster wall-clock) has no way to
turn the monolithic inline check off, so it would run twice.

## What Changes

- Add a `run_flake_check` input (boolean, default `true`) to `build.yml` and the
  orchestrator `ci.yml`.
- When `false`: the standalone `flake-check` job is skipped, and the OCI `build`
  job skips its inline **Flake check**, **Build coverage**, and **Codecov** steps
  — it only builds/pushes the OCI image. The consumer owns checks + coverage.
- Default `true` preserves today's behavior for every existing consumer.

## Capabilities

### New Capabilities
- None.

### Modified Capabilities
- `oci-build`: the build workflow's flake-check + coverage steps become
  conditional on `run_flake_check`; image build/push are unconditional.
- `reusable-ci`: the orchestrator gains the `run_flake_check` input, forwards it
  to `build.yml`, and gates the standalone `flake-check` job on it.

## Impact

- **Affected:** `.github/workflows/build.yml`, `.github/workflows/ci.yml`, and the
  `oci-build` + `reusable-ci` specs.
- **Consumers:** backward compatible — `run_flake_check` defaults to `true`.
- **I/O / network / memory:** none (CI control flow only). Enables a consumer to
  drop redundant work; no effect on any product runtime.

## Non-goals

- Implementing the fan-out matrix itself (lives in the consumer, e.g. ncps).
- Changing the OCI build, manifest, tag, or fork-safety logic.
