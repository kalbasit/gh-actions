# oci-build (delta: scope-flake-check-systems)

## MODIFIED Requirements

### Requirement: Reusable OCI build workflow

The repository SHALL provide a reusable workflow at `.github/workflows/build.yml` that, for
each system in `inputs.systems`, builds the OCI image via `nix build
.#packages.<system>.docker`, and — when `push_oci` is true and the build is not from a fork
PR — pushes the image to Docker Hub and ghcr.io. For each system that is in
`inputs.test_systems` — or for every system when `test_systems` is the empty array `"[]"`
(the default) or an empty string — the workflow SHALL additionally run `nix flake check -L`
and build+upload coverage to Codecov (using `codecov-action@v6`). A follow-up `oci-manifest` job SHALL
assemble a multi-arch manifest from the per-system images and push it.

#### Scenario: PR from the owner with `push_oci: true`

- **WHEN** invoked from a non-fork PR with `systems: '["x86_64-linux","aarch64-linux"]'`,
  `images: "kalbasit/foo"`, `push_oci: true`, and `test_systems: '[]'` (default)
- **THEN** the workflow SHALL run two matrix jobs (one per arch), each performing flake-check,
  coverage build, codecov upload, OCI image build, registry login, image push; SHALL then run
  `oci-manifest` to `docker manifest create` and `docker manifest push` a single multi-arch
  manifest per generated tag

#### Scenario: PR with `test_systems` restricting the integration suite

- **WHEN** invoked from a non-fork PR with `systems: '["x86_64-linux","aarch64-linux"]'` and
  `test_systems: '["x86_64-linux"]'`
- **THEN** the `x86_64-linux` leg SHALL run flake-check + coverage + codecov upload, while the
  `aarch64-linux` leg SHALL skip flake-check and coverage
- **AND** both legs SHALL still build (and, when pushing, push) their OCI image
- **AND** `oci-manifest` SHALL still assemble the multi-arch manifest from both per-arch images

#### Scenario: PR from a fork

- **WHEN** invoked from a fork PR
- **THEN** the workflow SHALL still run flake-check (on the `test_systems` subset, or every
  arch when `test_systems` is `"[]"`) and the OCI image build (read-only steps); SHALL skip
  codecov upload, registry login, push, and the `oci-manifest` job

#### Scenario: `oci: false` upstream

- **WHEN** the orchestrator's `oci` input is false
- **THEN** the orchestrator SHALL NOT invoke `build.yml`; standalone `flake-check` runs in
  its place
