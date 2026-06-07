## MODIFIED Requirements

### Requirement: Reusable CI orchestrator workflow

The repository SHALL expose a reusable workflow at `.github/workflows/ci.yml` invokable
cross-repo via `uses: kalbasit/gh-actions/.github/workflows/ci.yml@<ref>` that orchestrates
the full CI pipeline (filter, generate, flake checks or build, openspec-guard, final gate)
for any Nix-flake consumer. The workflow SHALL accept a `check_mode` input
(`single` | `matrix` | `none`, default `single`) selecting how `nix flake check` runs, and a
`coverage` input (boolean, default `false`) selecting whether a coverage job runs. The
workflow SHALL accept a `test_systems` input (JSON-array string, default `"[]"`) that scopes
checks and coverage to a subset of `systems`; `"[]"` runs them on every system. The
orchestrator SHALL forward `check_mode` and `test_systems` to `build.yml`, SHALL apply
`check_mode` to its standalone check jobs, and SHALL run the `coverage` job (when enabled)
itself rather than inside `build.yml`.

#### Scenario: Consumer with Go and no OCI images, default mode

- **WHEN** a consumer's `ci.yml` invokes `kalbasit/gh-actions/.github/workflows/ci.yml@<ref>`
  with `languages: '{"go":{...}}'`, `oci: false`, `cachix_cache: "foo"`, and `check_mode`
  omitted
- **THEN** the workflow SHALL run `filter`, `generate` (Go branch only), a single
  `flake-check` job (per system) running monolithic `nix flake check -L`, `openspec-guard`,
  and a final `ci` gate job
- **AND** SHALL NOT run the `check-matrix`/`checks` fan-out, nor any OCI-build, coverage,
  registry-login, or manifest steps

#### Scenario: Consumer with Go and OCI images enabled

- **WHEN** the same call is made with `oci: true`, `check_mode: single`,
  `systems: '["x86_64-linux","aarch64-linux"]'`, and `dockerhub_image: "kalbasit/foo"`
- **THEN** the workflow SHALL run `filter`, `generate`, `build` (per-arch matrix with the
  inline `nix flake check`, OCI image build, push, manifest), `openspec-guard`, and final
  `ci`
- **AND** SHALL NOT run a separate standalone `flake-check` job (it lives inside the build
  matrix)

#### Scenario: Consumer scopes checks with `test_systems`

- **WHEN** a consumer invokes the orchestrator with
  `systems: '["x86_64-linux","aarch64-linux"]'` and `test_systems: '["x86_64-linux"]'`
- **THEN** checks SHALL run only on the `x86_64-linux` leg (whether via the `build` matrix
  when `oci: true`, the standalone `flake-check` job in `single` mode, or the `check-matrix`
  fan-out in `matrix` mode)
- **AND** when `oci: true`, the `aarch64-linux` leg SHALL still build its OCI image

### Requirement: Inputs

The `ci.yml` workflow SHALL accept the following inputs:

| Name | Type | Required | Default | Purpose |
| --- | --- | --- | --- | --- |
| `cachix_cache` | string | yes | — | Cachix cache name |
| `languages` | string (JSON object) | yes | — | Per-language config; keys: `go`, future: `python`, `rust`, ... |
| `oci` | boolean | no | `false` | Enable OCI image build/push |
| `systems` | string (JSON array) | no | `["x86_64-linux","aarch64-linux"]` | Nix systems to build/check |
| `test_systems` | string (JSON array) | no | `"[]"` | Subset of `systems` on which to run checks + coverage; empty (or `""`) means all systems |
| `check_mode` | string (`single`\|`matrix`\|`none`) | no | `single` | How `nix flake check` runs: `single` = one monolithic check per system; `matrix` = fan each `.#checks.<system>` attribute out into its own parallel job; `none` = skip checks (OCI build/push unaffected) |
| `coverage` | boolean | no | `false` | When `true`, run a fork-safe `coverage` job that builds `.#<primary_package>.coverage` and uploads to Codecov |
| `primary_package` | string | when `oci: true` or `coverage: true` | `""` | Primary Nix package attr (used for `.#<pkg>.coverage` and `version.txt`) |
| `dockerhub_image` | string | when `oci: true` | — | Docker Hub image base (e.g., `kalbasit/foo`) |
| `dockerhub_username` | string | when `oci: true` | — | Docker Hub username |
| `push_oci` | boolean | no | `true` | Whether to push images to registries (false for fork PRs) |
| `openspec_path` | string | no | `openspec/changes` | Path searched for active OpenSpec changes |
| `extra_filters` | string (JSON object) | no | `{}` | Additional `dorny/paths-filter` rules merged into the built-in set |

#### Scenario: Missing required input

- **WHEN** a consumer omits `cachix_cache`
- **THEN** GitHub Actions SHALL fail the workflow call with a missing-input error before any
  job runs

#### Scenario: Defaults applied

- **WHEN** a consumer omits `systems`, `oci`, `check_mode`, and `coverage`
- **THEN** the workflow SHALL default `systems` to `["x86_64-linux","aarch64-linux"]`, `oci`
  to `false`, `check_mode` to `single`, and `coverage` to `false`

## ADDED Requirements

### Requirement: check_mode input

The `ci.yml` workflow SHALL accept a `check_mode` input: a string, not required, defaulting
to `single`, whose only valid values are `single`, `matrix`, and `none`. The workflow SHALL
reject any other value with a failing guard before checks run. `single` SHALL run the
existing monolithic `nix flake check -L` (the standalone `flake-check` job when `oci: false`,
or the inline check inside `build.yml` when `oci: true`). `matrix` SHALL fan the checks out
(see "Flake-check matrix mode"). `none` SHALL skip all inline checks while leaving any OCI
image build/push unaffected. The workflow SHALL forward `check_mode` to `build.yml`.

#### Scenario: Default keeps the single monolithic check

- **WHEN** a consumer omits `check_mode`
- **THEN** it SHALL default to `single` and CI SHALL run one `nix flake check -L` per system
  as before

#### Scenario: none skips checks but not the OCI build

- **WHEN** a consumer sets `check_mode: none` with `oci: true`
- **THEN** the orchestrator SHALL NOT run a standalone `flake-check` job nor the
  `check-matrix`/`checks` fan-out
- **AND** SHALL forward `check_mode: none` to `build.yml`, whose inline check SHALL be skipped
  while the OCI image is still built and pushed

#### Scenario: Invalid value fails fast

- **WHEN** a consumer sets `check_mode` to any value other than `single`, `matrix`, or `none`
- **THEN** the workflow SHALL fail with an explicit error rather than silently defaulting

### Requirement: Flake-check matrix mode

When `check_mode` is `matrix`, the `ci.yml` workflow SHALL run a `check-matrix` job that, for
each effective system (the `test_systems` array when non-empty, otherwise every system in
`systems`), enumerates check attribute names via
`nix eval .#checks.<system> --apply builtins.attrNames --json` and emits a single matrix
`include` output: a JSON array of `{system, check}` objects. A downstream `checks` job SHALL
consume that include list with `fail-fast: false`, select its runner per system
(`ubuntu-24.04-arm` for `aarch64-linux`, else `ubuntu-24.04`), and build each check with
`nix build ".#checks.$SYSTEM.$CHECK" -L`. The `checks` job SHALL pass the matrix `system` and
`check` values through `env:` and reference them as quoted shell variables, and SHALL NOT
interpolate `${{ matrix.* }}` into the `run:` script, to remain safe against template
injection from fork PRs. This matrix fan-out SHALL run in `ci.yml` for both `oci: false` and
`oci: true`; when `oci: true`, `build.yml` SHALL build the OCI image without an inline check.

#### Scenario: Matrix mode fans out per system × check

- **WHEN** a consumer sets `check_mode: matrix` with
  `systems: '["x86_64-linux","aarch64-linux"]'`, `test_systems: '[]'`, and a flake exposing
  checks `a` and `b` on both systems
- **THEN** `check-matrix` SHALL emit four `{system, check}` entries and `checks` SHALL run
  four parallel jobs, each building `nix build ".#checks.$SYSTEM.$CHECK" -L`

#### Scenario: Matrix mode honors test_systems

- **WHEN** a consumer sets `check_mode: matrix`,
  `systems: '["x86_64-linux","aarch64-linux"]'`, and `test_systems: '["x86_64-linux"]'`
- **THEN** `check-matrix` SHALL enumerate checks only for `x86_64-linux` and `checks` SHALL
  run no `aarch64-linux` check jobs

#### Scenario: Injection-safe check execution

- **WHEN** a fork PR exposes a check whose attribute name contains shell metacharacters
- **THEN** the `checks` job SHALL treat the name as data via an `env:` variable and SHALL NOT
  execute it as part of the `run:` template

#### Scenario: Flake with no checks is a no-op

- **WHEN** `check_mode: matrix` is set on a flake whose `.#checks.<system>` is empty
- **THEN** `check-matrix` SHALL emit an empty include list, `checks` SHALL run zero jobs, and
  the final `ci` gate SHALL still pass

### Requirement: Coverage upload

When `coverage` is `true`, the `ci.yml` workflow SHALL run a single `coverage` job that builds
`.#<primary_package>.coverage` on the primary system (the first of `test_systems`, else
`x86_64-linux`) and uploads the resulting `result-coverage` to Codecov via `codecov-action`.
The job SHALL require `primary_package` and the `codecov_token` secret, and SHALL be skipped on
fork PRs or when `codecov_token` is absent. When `coverage` is `false` (the default), no
coverage job SHALL run and `build.yml` SHALL NOT build or upload coverage.

#### Scenario: Coverage enabled on a non-fork build

- **WHEN** a consumer sets `coverage: true`, `primary_package: "foo"`, and provides
  `codecov_token` on a non-fork build
- **THEN** the `coverage` job SHALL build `.#foo.coverage` and upload `result-coverage` to
  Codecov

#### Scenario: Coverage disabled by default

- **WHEN** a consumer omits `coverage`
- **THEN** no `coverage` job SHALL run and neither `ci.yml` nor `build.yml` SHALL upload to
  Codecov

#### Scenario: Coverage skipped on fork PRs

- **WHEN** `coverage: true` and the build is a fork PR (no `codecov_token`)
- **THEN** the `coverage` job SHALL be skipped and the final `ci` gate SHALL treat the skip as
  success

## REMOVED Requirements

### Requirement: run_flake_check input

**Reason**: Replaced by the `check_mode` enum, which expresses the same skip behavior
(`check_mode: none`) alongside the new `single` and `matrix` modes; the standalone boolean
could not represent the matrix option.

**Migration**: Consumers that set `run_flake_check: true` SHALL use `check_mode: single` (the
default, so the input may simply be dropped); consumers that set `run_flake_check: false` to
fan checks out themselves SHALL use `check_mode: matrix` (to fan out via the reusable workflow)
or `check_mode: none` (to skip inline checks). Inline coverage previously implied by
`run_flake_check: true` SHALL be requested explicitly via `coverage: true`.
