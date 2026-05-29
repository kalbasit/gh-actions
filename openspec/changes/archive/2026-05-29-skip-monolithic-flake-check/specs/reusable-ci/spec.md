# reusable-ci (delta: skip-monolithic-flake-check)

## ADDED Requirements

### Requirement: run_flake_check input

The `ci.yml` workflow SHALL accept a `run_flake_check` input: a boolean, not required,
defaulting to `true`. When `true`, CI runs `nix flake check` + coverage inline (the standalone
`flake-check` job when `oci: false`, or the inline steps inside the `build` job when
`oci: true`). When `false`, the standalone `flake-check` job SHALL be skipped and the value
SHALL be forwarded to `build.yml` so its inline check + coverage steps are skipped too — for
consumers that fan the checks out into their own parallel matrix. The OCI image build/push is
unaffected by this input.

#### Scenario: Default keeps inline checks

- **WHEN** a consumer omits `run_flake_check`
- **THEN** it SHALL default to `true` and CI SHALL run the flake check + coverage inline as
  before

#### Scenario: Disabled skips inline checks and forwards to build

- **WHEN** a consumer sets `run_flake_check: false` with `oci: true`
- **THEN** the orchestrator SHALL NOT run a standalone `flake-check` job
- **AND** SHALL pass `run_flake_check: false` to `build.yml`, whose inline flake-check and
  coverage steps SHALL then be skipped while the OCI image is still built and pushed
