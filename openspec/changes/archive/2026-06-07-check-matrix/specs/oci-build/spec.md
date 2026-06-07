## MODIFIED Requirements

### Requirement: Reusable OCI build workflow

The repository SHALL provide a reusable workflow at `.github/workflows/build.yml` that, for
each system in `inputs.systems`, builds the OCI image via `nix build
.#packages.<system>.docker`, and — when `push_oci` is true and the build is not from a fork
PR — pushes the image to Docker Hub and ghcr.io. The workflow SHALL accept a `check_mode`
input (`single` | `matrix` | `none`, default `single`) forwarded by the orchestrator. When
`check_mode` is `single`, then for each system that is in `inputs.test_systems` — or for every
system when `test_systems` is the empty array `"[]"` (the default) or an empty string — the
workflow SHALL additionally run `nix flake check -L` inline. When `check_mode` is `matrix` or
`none`, the workflow SHALL NOT run any inline `nix flake check` (in `matrix` mode the checks
fan out in `ci.yml` instead). The workflow SHALL NOT build coverage or upload to Codecov in any
mode; coverage is owned by the orchestrator's `coverage` job. A follow-up `oci-manifest` job
SHALL assemble a multi-arch manifest from the per-system images and push it.

The workflow SHALL validate registry inputs only when it would actually push (`push_oci` is
true): when pushing, it SHALL require that `dockerhub_image` and `dockerhub_username` are
either both set or both empty, and that at least one registry (Docker Hub or ghcr.io) is
enabled. When `push_oci` is false, the workflow SHALL NOT fail on a partially-set
`dockerhub_image`/`dockerhub_username` pair, because no registry login, push, or manifest step
runs in that case.

#### Scenario: check_mode none skips the inline check

- **WHEN** invoked with `check_mode: none`
- **THEN** the workflow SHALL NOT run `nix flake check`
- **AND** it SHALL still build (and, when pushing, push) the OCI image per system and assemble
  the multi-arch manifest

#### Scenario: check_mode matrix skips the inline check (checks fan out upstream)

- **WHEN** invoked with `check_mode: matrix`
- **THEN** the workflow SHALL NOT run an inline `nix flake check`; the per-check fan-out runs in
  `ci.yml`
- **AND** it SHALL still build (and, when pushing, push) the OCI image per system and assemble
  the multi-arch manifest

#### Scenario: PR from the owner with `push_oci: true`

- **WHEN** invoked from a non-fork PR with `systems: '["x86_64-linux","aarch64-linux"]'`,
  `dockerhub_image: "kalbasit/foo"`, `push_oci: true`, `check_mode: single`, and
  `test_systems: '[]'` (default)
- **THEN** the workflow SHALL run two matrix jobs (one per arch), each performing the inline
  flake-check, OCI image build, registry login, and image push; SHALL then run `oci-manifest`
  to `docker manifest create` and `docker manifest push` a single multi-arch manifest per
  generated tag
- **AND** SHALL NOT build coverage or upload to Codecov

#### Scenario: PR with `test_systems` restricting the inline check

- **WHEN** invoked from a non-fork PR with `systems: '["x86_64-linux","aarch64-linux"]'`,
  `check_mode: single`, and `test_systems: '["x86_64-linux"]'`
- **THEN** the `x86_64-linux` leg SHALL run the inline flake-check, while the `aarch64-linux`
  leg SHALL skip it
- **AND** both legs SHALL still build (and, when pushing, push) their OCI image
- **AND** `oci-manifest` SHALL still assemble the multi-arch manifest from both per-arch images

#### Scenario: PR from a fork

- **WHEN** invoked from a fork PR with `check_mode: single` (the caller forces
  `push_oci: false`, and `dockerhub_username` is empty because the variable is withheld from
  fork PRs while `dockerhub_image` remains a literal)
- **THEN** the workflow SHALL NOT fail registry-input validation despite the asymmetric
  `dockerhub_image` (set) / `dockerhub_username` (empty) pair
- **AND** it SHALL still run the inline flake-check (on the `test_systems` subset, or every arch
  when `test_systems` is `"[]"`) and the OCI image build (read-only steps)
- **AND** it SHALL skip registry login, push, and the `oci-manifest` job

#### Scenario: `oci: false` upstream

- **WHEN** the orchestrator's `oci` input is false
- **THEN** the orchestrator SHALL NOT invoke `build.yml`; the standalone `flake-check` job
  (`single` mode) or `check-matrix`/`checks` fan-out (`matrix` mode) runs in its place
