# oci-build Specification

## Purpose
TBD - created by archiving change shared-workflows. Update Purpose after archive.
## Requirements

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

### Requirement: Per-arch runner selection

The matrix SHALL select runner labels by system: `ubuntu-24.04-arm` for `aarch64-linux`,
`ubuntu-24.04` for all other systems.

#### Scenario: Matrix expansion

- **WHEN** `systems` is `["x86_64-linux","aarch64-linux"]`
- **THEN** the x86_64 job SHALL run on `ubuntu-24.04` and the aarch64 job on
  `ubuntu-24.04-arm`

### Requirement: Image tag generation

The workflow SHALL use `docker/metadata-action@v6` to generate per-arch tags using the
following rules (carried over from current ncps/signal-api-receiver build.yml):

- `type=ref,event=branch`
- `type=ref,event=pr`
- `type=semver,pattern={{raw}}`
- `type=semver,pattern=v{{version}}`
- `type=semver,pattern=v{{major}}.{{minor}}`
- `type=semver,pattern=v{{major}}`
- `type=sha`

Image bases SHALL be `${{ inputs.images }}` and `ghcr.io/${{ github.repository }}`.

#### Scenario: Tag pushed on a `v1.2.3` git tag

- **WHEN** the workflow runs on a push of git tag `v1.2.3`
- **THEN** tags including `v1.2.3`, `v1`, `v1.2`, and a `sha-<short>` tag SHALL be produced
  for each registry base

### Requirement: Multi-arch manifest

The follow-up `oci-manifest` job SHALL, for each tag emitted by the per-arch metadata step,
create a Docker manifest list combining the per-arch images (`<tag>-<system>`) and push it.

#### Scenario: Manifest creation

- **WHEN** two per-arch images exist for tag `kalbasit/foo:v1.2.3`
- **THEN** the job SHALL run `docker manifest create kalbasit/foo:v1.2.3
  kalbasit/foo:v1.2.3-x86_64-linux kalbasit/foo:v1.2.3-aarch64-linux` and then `docker
  manifest push kalbasit/foo:v1.2.3`

### Requirement: Version file injection

Before flake-check, the workflow SHALL check whether `github.ref_name` matches the semver
pattern `^v[0-9]+\.[0-9]+\.[0-9]+(-[0-9A-Za-z.-]+)?(\+[0-9A-Za-z.-]+)?$`; if it matches, the
workflow MUST write that ref name into `nix/packages/<primary-package>/version.txt` so the
Nix build embeds the correct version.

#### Scenario: Branch build

- **WHEN** the build runs on a branch (not a semver tag)
- **THEN** the workflow SHALL NOT write `version.txt`; the default version baked into
  `default.nix` SHALL be used
