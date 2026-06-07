## MODIFIED Requirements

### Requirement: Reusable OCI build workflow

The repository SHALL provide a reusable workflow at `.github/workflows/build.yml` that, for
each system in `inputs.systems`, builds the OCI image via `nix build
.#packages.<system>.docker`, and — when `push_oci` is true and the build is not from a fork
PR — pushes the image to Docker Hub and ghcr.io. When `inputs.run_flake_check` is true (the
default), then for each system that is in `inputs.test_systems` — or for every system when
`test_systems` is the empty array `"[]"` (the default) or an empty string — the workflow SHALL
additionally run `nix flake check -L` and build+upload coverage to Codecov (using
`codecov-action@v6`). When `run_flake_check` is false, the workflow SHALL skip the flake-check
and coverage steps entirely and only build/push the image. A follow-up `oci-manifest` job SHALL
assemble a multi-arch manifest from the per-system images and push it.

The workflow SHALL validate registry inputs only when it would actually push (`push_oci` is
true): when pushing, it SHALL require that `dockerhub_image` and `dockerhub_username` are
either both set or both empty, and that at least one registry (Docker Hub or ghcr.io) is
enabled. When `push_oci` is false, the workflow SHALL NOT fail on a partially-set
`dockerhub_image`/`dockerhub_username` pair, because no registry login, push, or manifest step
runs in that case.

#### Scenario: run_flake_check false skips the inline check

- **WHEN** invoked with `run_flake_check: false`
- **THEN** the workflow SHALL NOT run `nix flake check`, build coverage, or upload to Codecov
- **AND** it SHALL still build (and, when pushing, push) the OCI image per system and assemble
  the multi-arch manifest

#### Scenario: PR from the owner with `push_oci: true`

- **WHEN** invoked from a non-fork PR with `systems: '["x86_64-linux","aarch64-linux"]'`,
  `images: "kalbasit/foo"`, `push_oci: true`, `run_flake_check: true`, and
  `test_systems: '[]'` (default)
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

- **WHEN** invoked from a fork PR with `run_flake_check: true` (the caller forces
  `push_oci: false`, and `dockerhub_username` is empty because the variable is withheld from
  fork PRs while `dockerhub_image` remains a literal)
- **THEN** the workflow SHALL NOT fail registry-input validation despite the asymmetric
  `dockerhub_image` (set) / `dockerhub_username` (empty) pair
- **AND** it SHALL still run flake-check (on the `test_systems` subset, or every arch when
  `test_systems` is `"[]"`) and the OCI image build (read-only steps)
- **AND** it SHALL skip codecov upload, registry login, push, and the `oci-manifest` job

#### Scenario: `oci: false` upstream

- **WHEN** the orchestrator's `oci` input is false
- **THEN** the orchestrator SHALL NOT invoke `build.yml`; standalone `flake-check` runs in
  its place
