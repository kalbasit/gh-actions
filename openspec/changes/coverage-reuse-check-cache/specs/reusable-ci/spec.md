## MODIFIED Requirements

### Requirement: Coverage upload

When `coverage` is `true`, the `ci.yml` workflow SHALL run a single `coverage` job that builds
`.#<primary_package>.coverage` on the primary system (the first of `test_systems`, else the
first of `systems`) and uploads the resulting `result-coverage` to Codecov via `codecov-action`.
The job SHALL require `primary_package` and the `codecov_token` secret, and SHALL be skipped on
fork PRs or when `codecov_token` is absent. When `coverage` is `false` (the default), no
coverage job SHALL run and `build.yml` SHALL NOT build or upload coverage.

The `coverage` job SHALL run only after the check-producing jobs that populate the Cachix binary
cache — `check-mode-guard`, `flake-check`, `check-matrix`, `checks`, and `build` — have completed,
so that any test/cohort derivations shared between those jobs and the coverage derivation are
served from Cachix instead of being rebuilt by the coverage job. Because those dependencies are
mode-dependent and several are skipped in any given configuration (e.g. `flake-check` is skipped
in `matrix` mode; `build` is skipped when `oci: false`), the `coverage` job's gating SHALL be
written so a skipped dependency does not cascade-skip coverage: the job SHALL still run when its
dependencies are skipped or successful, and SHALL continue to honour the `inputs.coverage` toggle
and the fork-PR / missing-`codecov_token` skip. The job SHALL NOT run when the workflow is
cancelled.

#### Scenario: Coverage enabled on a non-fork build

- **WHEN** a consumer sets `coverage: true`, `primary_package: "foo"`, and provides
  `codecov_token` on a non-fork build
- **THEN** the `coverage` job SHALL build `.#foo.coverage` and upload `result-coverage` to
  Codecov

#### Scenario: Coverage reuses the check jobs' Cachix artifacts

- **WHEN** `coverage: true` and the active `check_mode` builds the coverage derivation's
  dependencies in its check-producing jobs (e.g. `check_mode: matrix` building the cohort
  derivations in `checks`)
- **THEN** the `coverage` job SHALL start only after those jobs complete, so the shared
  derivations are restored from Cachix and the coverage job does not rebuild them from scratch

#### Scenario: Coverage runs despite mode-dependent dependency skips

- **WHEN** `coverage: true` and one or more of the coverage job's dependencies are skipped for
  the active configuration (e.g. `flake-check` skipped in `matrix` mode, or `build` skipped when
  `oci: false`)
- **THEN** the `coverage` job SHALL still run (the skip SHALL NOT cascade-skip it) and the final
  `ci` gate SHALL treat its result as required

#### Scenario: Coverage disabled by default

- **WHEN** a consumer omits `coverage`
- **THEN** no `coverage` job SHALL run and neither `ci.yml` nor `build.yml` SHALL upload to
  Codecov

#### Scenario: Coverage skipped on fork PRs

- **WHEN** `coverage: true` and the build is a fork PR (no `codecov_token`)
- **THEN** the `coverage` job SHALL be skipped and the final `ci` gate SHALL treat the skip as
  success
