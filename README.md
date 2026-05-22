# gh-actions

Reusable GitHub Actions workflows for Nix-flake projects under
[github.com/kalbasit](https://github.com/kalbasit).

Consumers (`ncps`, `swm`, `signal-api-receiver`) call these workflows via
`workflow_call` and ship a thin caller in their own `.github/workflows/ci.yml`.

## Workflows

| File | Purpose |
| --- | --- |
| `.github/workflows/ci.yml` | Top-level orchestrator. Wires filter → generate → flake-check or OCI build → openspec-guard → final gate. |
| `.github/workflows/generate.yml` | Language-aware dependency-hash regenerator. Updates Nix `vendorHash` (and future Python/Rust equivalents) and auto-commits. |
| `.github/workflows/build.yml` | Per-arch flake-check + coverage + OCI image build/push + multi-arch manifest. Called by `ci.yml` when `oci: true`. |
| `.github/workflows/openspec-guard.yml` | Fails CI when active (non-archived) OpenSpec changes exist under `openspec/changes/`. |

## Quick start

```yaml
# .github/workflows/ci.yml in your repo
name: CI
on:
  pull_request:
    branches: [main, "release-*"]
  push:
    branches: [main, "release-*"]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  ci:
    uses: kalbasit/gh-actions/.github/workflows/ci.yml@v0.1.0
    with:
      cachix_cache: your-cache
      languages: |
        {"go":{"packages":["your-app"]}}
      oci: true
      primary_package: your-app
      images: yourhubuser/your-app
      dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
    secrets:
      cachix_auth_token: ${{ secrets.CACHIX_AUTH_TOKEN }}
      gha_pat_token: ${{ secrets.GHA_PAT_TOKEN }}
      codecov_token: ${{ secrets.CODECOV_TOKEN }}
      dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

## Inputs (`ci.yml`)

| Name | Type | Required | Default | Purpose |
| --- | --- | --- | --- | --- |
| `cachix_cache` | string | yes | — | Cachix cache name |
| `languages` | string (JSON object) | yes | — | Per-language config (see below) |
| `oci` | boolean | no | `false` | Enable OCI image build/push |
| `systems` | string (JSON array) | no | `["x86_64-linux","aarch64-linux"]` | Nix systems to build/check |
| `primary_package` | string | when `oci: true` | — | Package attr for the coverage build (`.#<pkg>.coverage`) and version.txt path |
| `images` | string | when `oci: true` | — | Image base name (e.g., `yourhubuser/foo`) |
| `dockerhub_username` | string | when `oci: true` | — | Docker Hub username |
| `push_oci` | boolean | no | `true` | Push to registries (auto-disabled on fork PRs) |
| `openspec_path` | string | no | `openspec/changes` | Path scanned by openspec-guard |
| `extra_filters` | string (JSON object) | no | `{}` | Extra `dorny/paths-filter` rules merged into the built-in set |

### `languages` shape

A JSON-encoded object keyed by language identifier:

```json
{
  "go": {"packages": ["myapp", "myapp-plugin-x"]}
}
```

Today only `go` is implemented. Future languages (`python`, `rust`, `node`) will plug
in as additional keys with their own per-language config — no input-shape change.

### Required secrets

| Secret | Required | Purpose |
| --- | --- | --- |
| `cachix_auth_token` | always | Cachix push token |
| `gha_pat_token` | when `generate` runs | PAT used by `git-auto-commit-action` to push back vendor-hash updates |
| `codecov_token` | when `oci: true` | Codecov upload token |
| `dockerhub_token` | when `oci: true` | Docker Hub registry credential (also used to publish to ghcr.io alongside `GITHUB_TOKEN`) |

## Consumer examples

### swm (no OCI, multiple Go packages)

```yaml
jobs:
  ci:
    uses: kalbasit/gh-actions/.github/workflows/ci.yml@v0.1.0
    with:
      cachix_cache: swm
      oci: false
      languages: |
        {"go":{"packages":[
          "swm",
          "swm-plugin-forge-github",
          "swm-plugin-picker-fzf",
          "swm-plugin-session-tmux",
          "swm-plugin-vcs-git"
        ]}}
    secrets:
      cachix_auth_token: ${{ secrets.CACHIX_AUTH_TOKEN }}
      gha_pat_token: ${{ secrets.GHA_PAT_TOKEN }}
```

### ncps (OCI + extra paths-filter for database/docs sibling jobs)

```yaml
jobs:
  ci:
    uses: kalbasit/gh-actions/.github/workflows/ci.yml@v0.1.0
    with:
      cachix_cache: ncps
      oci: true
      primary_package: ncps
      images: kalbasit/ncps
      dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
      languages: |
        {"go":{"packages":["ncps"]}}
      extra_filters: |
        {
          "database": ["db/migrations/**", "db/query.*.sql", "sqlc.yml", "pkg/database/**"],
          "docs": ["docs/**"]
        }
    secrets:
      cachix_auth_token: ${{ secrets.CACHIX_AUTH_TOKEN }}
      gha_pat_token: ${{ secrets.GHA_PAT_TOKEN }}
      codecov_token: ${{ secrets.CODECOV_TOKEN }}
      dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}

  # ncps-specific sibling jobs (database codegen, docs deploy) stay in ncps's repo
  # and depend on the filter outputs exposed by the reusable workflow.
```

### signal-api-receiver (OCI, single package)

```yaml
jobs:
  ci:
    uses: kalbasit/gh-actions/.github/workflows/ci.yml@v0.1.0
    with:
      cachix_cache: signal-api-receiver
      oci: true
      primary_package: signal-api-receiver
      images: kalbasit/signal-api-receiver
      dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
      languages: |
        {"go":{"packages":["signal-api-receiver"]}}
    secrets:
      cachix_auth_token: ${{ secrets.CACHIX_AUTH_TOKEN }}
      gha_pat_token: ${{ secrets.GHA_PAT_TOKEN }}
      codecov_token: ${{ secrets.CODECOV_TOKEN }}
      dockerhub_token: ${{ secrets.DOCKERHUB_TOKEN }}
```

## Versioning

- Pin to a tag (`@v0.1.0`, `@v1`) or a SHA. Avoid `@main` for production CI.
- A floating `v1` tag tracks the latest non-breaking release inside major 1.

## Conventions

- When the source workflows disagreed on how to do something, the newer repo won.
  Current precedence: **SWM > ncps > signal-api-receiver**.
- OCI is the preferred term over "Docker" when referring to images.
- Language support is additive: new languages plug in via the `languages` input
  without restructuring the workflow.
