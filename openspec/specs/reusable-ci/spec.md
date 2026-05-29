# reusable-ci Specification

## Purpose
TBD - created by archiving change shared-workflows. Update Purpose after archive.
## Requirements

### Requirement: Reusable CI orchestrator workflow

The repository SHALL expose a reusable workflow at `.github/workflows/ci.yml` invokable
cross-repo via `uses: kalbasit/gh-actions/.github/workflows/ci.yml@<ref>` that orchestrates
the full CI pipeline (filter, generate, flake-check or build, openspec-guard, final gate)
for any Nix-flake consumer. The workflow SHALL accept a `test_systems` input (JSON-array
string, default `"[]"`) that scopes the integration suite (`nix flake check`) and coverage to
a subset of `systems`; `"[]"` runs them on every system. The orchestrator SHALL forward
`test_systems` to `build.yml` and SHALL apply the same scoping to its standalone
`flake-check` job.

#### Scenario: Consumer with Go and no OCI images

- **WHEN** a consumer's `ci.yml` invokes `kalbasit/gh-actions/.github/workflows/ci.yml@<ref>`
  with `languages: '{"go":{"packages":["foo"]}}'`, `oci: false`, and `cachix_cache: "foo"`
- **THEN** the workflow SHALL run `filter`, `generate` (Go branch only),
  `flake-check` (per system), `openspec-guard`, and a final `ci` gate job
- **AND** SHALL NOT run any OCI-build, codecov, registry-login, or manifest steps

#### Scenario: Consumer with Go and OCI images enabled

- **WHEN** the same call is made with `oci: true`, `systems: '["x86_64-linux","aarch64-linux"]'`,
  and `images: "kalbasit/foo"`
- **THEN** the workflow SHALL run `filter`, `generate`, `build` (per-arch matrix with
  flake-check, coverage, codecov upload, OCI image build, push, manifest), `openspec-guard`,
  and final `ci`
- **AND** SHALL NOT run a separate `flake-check` job (it lives inside the build matrix)

#### Scenario: Consumer scopes the integration suite with `test_systems`

- **WHEN** a consumer invokes the orchestrator with
  `systems: '["x86_64-linux","aarch64-linux"]'` and `test_systems: '["x86_64-linux"]'`
- **THEN** flake-check and coverage SHALL run only on the `x86_64-linux` leg (whether via the
  `build` matrix when `oci: true` or the standalone `flake-check` job when `oci: false`)
- **AND** when `oci: true`, the `aarch64-linux` leg SHALL still build its OCI image

### Requirement: test_systems input

The `ci.yml` workflow SHALL accept a `test_systems` input: a JSON-array string, not required,
defaulting to `"[]"`. It names the subset of `systems` on which `nix flake check` and coverage
run; an empty array — or an empty string — MUST be treated as "all systems". The workflow
SHALL forward this value unchanged to `build.yml`.

#### Scenario: Default omitted

- **WHEN** a consumer omits `test_systems`
- **THEN** the workflow SHALL default it to `"[]"` and run flake-check + coverage on every
  system in `systems`

#### Scenario: Forwarded to build.yml

- **WHEN** `oci: true` and a consumer sets `test_systems: '["x86_64-linux"]'`
- **THEN** the orchestrator SHALL pass `test_systems: '["x86_64-linux"]'` to `build.yml`

### Requirement: Inputs

The `ci.yml` workflow SHALL accept the following inputs:

| Name | Type | Required | Default | Purpose |
| --- | --- | --- | --- | --- |
| `cachix_cache` | string | yes | — | Cachix cache name |
| `languages` | string (JSON object) | yes | — | Per-language config; keys: `go`, future: `python`, `rust`, ... |
| `oci` | boolean | no | `false` | Enable OCI image build/push |
| `systems` | string (JSON array) | no | `["x86_64-linux","aarch64-linux"]` | Nix systems to build/check |
| `images` | string | when `oci: true` | — | Image name base (e.g., `kalbasit/foo`) |
| `dockerhub_username` | string | when `oci: true` | — | Docker Hub username |
| `push_oci` | boolean | no | `true` | Whether to push images to registries (false for fork PRs) |
| `openspec_path` | string | no | `openspec/changes` | Path searched for active OpenSpec changes |
| `extra_filters` | string (JSON object) | no | `{}` | Additional `dorny/paths-filter` rules merged into the built-in set |

#### Scenario: Missing required input

- **WHEN** a consumer omits `cachix_cache`
- **THEN** GitHub Actions SHALL fail the workflow call with a missing-input error before any
  job runs

#### Scenario: Defaults applied

- **WHEN** a consumer omits `systems` and `oci`
- **THEN** the workflow SHALL default `systems` to `["x86_64-linux","aarch64-linux"]` and
  `oci` to `false`

### Requirement: Final CI gate

The workflow SHALL include a terminal `ci` job that depends on all preceding jobs,
runs with `if: always()`, and SHALL enumerate each upstream `needs.<job>.result`
individually rather than iterating `toJSON(needs)`.

#### Scenario: All upstream jobs succeed or skip

- **WHEN** every `needs.<job>.result` is `success` or `skipped`
- **THEN** the `ci` job SHALL exit 0 with message "All jobs passed"

#### Scenario: An upstream job fails

- **WHEN** any `needs.<job>.result` is `failure` or `cancelled`
- **THEN** the `ci` job SHALL emit `::error::Job failed with result: <result>` naming the
  specific failing job and exit non-zero

### Requirement: Fork safety

Jobs requiring write tokens (`generate`, OCI registry push, codecov upload) SHALL be skipped
when `github.event.pull_request.head.repo.fork == true`, and the final gate SHALL treat
those `skipped` results as success.

#### Scenario: Pull request from a fork

- **WHEN** a pull request opens from a forked repository
- **THEN** `generate` SHALL be skipped, OCI push steps SHALL be skipped, codecov upload SHALL
  be skipped
- **AND** the `ci` gate SHALL still exit 0 if remaining jobs succeed

### Requirement: Trigger contract

The workflow SHALL be invoked via `workflow_call` only; consumers wire their own
`pull_request` / `push` / `workflow_dispatch` triggers in their thin caller.

#### Scenario: Direct trigger attempt

- **WHEN** something tries to invoke `ci.yml` outside `workflow_call`
- **THEN** GitHub Actions SHALL reject the invocation (no other trigger declared)

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
