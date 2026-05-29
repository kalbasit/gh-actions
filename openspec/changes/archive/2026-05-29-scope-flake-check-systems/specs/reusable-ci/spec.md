# reusable-ci (delta: scope-flake-check-systems)

## MODIFIED Requirements

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

## ADDED Requirements

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
