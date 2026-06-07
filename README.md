# gh-actions

Reusable GitHub Actions workflows for Nix-flake projects under
[github.com/kalbasit](https://github.com/kalbasit).

Consumers (`ncps`, `swm`, `signal-api-receiver`) call these workflows via
`workflow_call` and ship a thin caller in their own `.github/workflows/ci.yml`.

## Workflows

| File | Purpose |
| --- | --- |
| `.github/workflows/ci.yml` | Top-level orchestrator. Wires filter → generate → flake checks (single/matrix) + optional coverage or OCI build → openspec-guard → final gate. |
| `.github/workflows/generate.yml` | Language-aware dependency-hash regenerator. Updates Nix `vendorHash` (and future Python/Rust equivalents) and auto-commits. |
| `.github/workflows/build.yml` | Per-arch inline flake-check (`single` mode) + OCI image build/push + multi-arch manifest. Called by `ci.yml` when `oci: true`. |
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
        {"go":{"targets":[{"attr":"your-app","file":"nix/packages/your-app/default.nix"}]}}
      oci: true
      primary_package: your-app
      dockerhub_image: yourhubuser/your-app
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
| `test_systems` | string (JSON array) | no | `"[]"` | Subset of `systems` on which to run checks + coverage; empty (or `""`) means all systems |
| `check_mode` | string (`single`/`matrix`/`none`) | no | `single` | How `nix flake check` runs: `single` = one monolithic check per system (good for quick repos); `matrix` = fan each `.#checks.<system>` attribute out into its own parallel job, scoped by `test_systems` (good for large projects); `none` = skip inline checks (OCI build/push unaffected) |
| `coverage` | boolean | no | `false` | Run a fork-safe coverage job that builds `.#<primary_package>.coverage` and uploads to Codecov. Requires `primary_package` + `codecov_token` |
| `primary_package` | string | when `oci: true` or `coverage: true` | — | Package attr for the coverage build (`.#<pkg>.coverage`) and version.txt path |
| `dockerhub_image` | string | when pushing to Docker Hub | `""` | Docker Hub image base name (e.g., `yourhubuser/foo`). Empty disables Docker Hub. |
| `dockerhub_username` | string | when pushing to Docker Hub | `""` | Docker Hub username. Must be set alongside `dockerhub_image`. |
| `ghcr_enabled` | boolean | no | `true` | Push images to a GitHub Container Registry image. |
| `ghcr_image` | string | no | `""` | Optional override for the ghcr.io base name. Empty falls back to `ghcr.io/<repo>`. |
| `push_oci` | boolean | no | `true` | Push to registries (auto-disabled on fork PRs) |
| `openspec_path` | string | no | `openspec/changes` | Path scanned by openspec-guard |
| `extra_filters` | string (JSON object) | no | `{}` | Extra `dorny/paths-filter` rules merged into the built-in set |

When `oci: true`, **at least one** of (Docker Hub via `dockerhub_image`+`dockerhub_username`, ghcr.io via `ghcr_enabled`) must resolve to an enabled registry, or the build fails up-front with a clear error.

### `languages` shape

A JSON-encoded object keyed by language identifier. Each language's value declares an
explicit list of **targets**:

```json
{
  "go": {
    "targets": [
      {"attr": "myapp", "file": "nix/packages/myapp/default.nix"},
      {"attr": "checks.x86_64-linux.myapp-drift-check", "file": "nix/checks/flake-module.nix"}
    ]
  }
}
```

- `attr` is the bare Nix attribute path. The language module appends a suffix
  (`.goModules` for Go) when invoking `nix build`.
- `file` is the path of the file containing the `vendorHash = "...";` line that the
  workflow sed-replaces. There is no convention — both fields are explicit.

Today only `go` is implemented. Future languages (`python`, `rust`, `node`) will plug
in as additional keys with the same `targets` shape — adding a language doesn't change
the input contract.

### Registries

Each registry is independently opt-in:

- **Docker Hub** is enabled when both `dockerhub_image` and `dockerhub_username` are non-empty.
- **ghcr.io** is enabled when `ghcr_enabled: true` (default). The image base defaults to
  `ghcr.io/<repo>`; pass `ghcr_image: ghcr.io/someorg/foo` to override.

Common configurations:

```yaml
# ghcr.io only, default repo
ghcr_enabled: true              # implicit default

# Docker Hub only
dockerhub_image: user/foo
dockerhub_username: user
ghcr_enabled: false

# Both, with a custom ghcr base
dockerhub_image: user/foo
dockerhub_username: user
ghcr_image: ghcr.io/someorg/foo
```

### Required secrets

| Secret | Required | Purpose |
| --- | --- | --- |
| `cachix_auth_token` | always | Cachix push token |
| `gha_pat_token` | when `generate` runs | PAT used by `git-auto-commit-action` to push back vendor-hash updates |
| `codecov_token` | when `coverage: true` | Codecov upload token |
| `dockerhub_token` | when Docker Hub is enabled | Docker Hub registry credential. ghcr.io uses the workflow's `GITHUB_TOKEN`. |

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
        {"go":{"targets":[
          {"attr":"swm",                     "file":"nix/packages/swm/default.nix"},
          {"attr":"swm-plugin-forge-github", "file":"nix/packages/swm-plugin-forge-github/default.nix"},
          {"attr":"swm-plugin-picker-fzf",   "file":"nix/packages/swm-plugin-picker-fzf/default.nix"},
          {"attr":"swm-plugin-session-tmux", "file":"nix/packages/swm-plugin-session-tmux/default.nix"},
          {"attr":"swm-plugin-vcs-git",      "file":"nix/packages/swm-plugin-vcs-git/default.nix"}
        ]}}
    secrets:
      cachix_auth_token: ${{ secrets.CACHIX_AUTH_TOKEN }}
      gha_pat_token: ${{ secrets.GHA_PAT_TOKEN }}
```

### ncps (OCI + matrix checks + coverage + extra paths-filter for sibling jobs)

`check_mode: matrix` fans each `.#checks.<system>` attribute out into its own parallel
job (here scoped to `x86_64-linux` via `test_systems`), and `coverage: true` builds
`.#ncps.coverage` and uploads it to Codecov — so the consumer no longer hand-rolls those
jobs.

```yaml
jobs:
  ci:
    uses: kalbasit/gh-actions/.github/workflows/ci.yml@v0.1.0
    with:
      cachix_cache: ncps
      oci: true
      primary_package: ncps
      dockerhub_image: kalbasit/ncps
      dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
      test_systems: '["x86_64-linux"]'
      check_mode: matrix
      coverage: true
      languages: |
        {"go":{"targets":[
          {"attr":"ncps",                                         "file":"nix/packages/ncps/default.nix"},
          {"attr":"checks.x86_64-linux.ent-codegen-drift-check",  "file":"nix/checks/flake-module.nix"}
        ]}}
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
      dockerhub_image: kalbasit/signal-api-receiver
      dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}
      languages: |
        {"go":{"targets":[{"attr":"signal-api-receiver","file":"nix/packages/signal-api-receiver/default.nix"}]}}
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

## Renovate

Every third-party action in this repo's workflows is pinned to a full commit SHA with a
trailing `# vN.N.N` comment. [Renovate](https://github.com/apps/renovate) keeps both
the SHA and the comment current, with policies declared in
[`.github/renovate.json5`](.github/renovate.json5):

- **Single weekly PR**: all GitHub-Action digest bumps land in one grouped PR
  scheduled before 6am on Monday. Vulnerability alerts bypass the schedule.
- **Auto-merge for trusted namespaces**: digest-only updates from
  `actions/`, `cachix/`, `codecov/`, `docker/`, `dorny/`, `stefanzweifel/`, and
  `DeterminateSystems/` auto-merge on green CI. Anything else opens a PR requiring
  human review.
- **Major bumps stay manual**: `v6 → v7` always requires a human merge regardless
  of namespace.
- **CI gate**: `.github/workflows/_lint-workflows.yml` runs `actionlint` on every PR
  to `main` — this is what blocks auto-merge when an upgrade breaks something.

The Renovate GitHub App must be installed on the repo for the config to take effect.
Install it once at [github.com/apps/renovate](https://github.com/apps/renovate).

**Branch protection note:** if/when `main` gets branch protection, mark
`actionlint` (from `_lint-workflows.yml`) as a required status check and either set
review requirements to zero or explicitly allow `renovate[bot]` to bypass them —
otherwise `platformAutomerge` won't actually merge. Renovate PRs trigger
`pull_request:` workflows regardless of protection state, so the lint check itself
needs no extra wiring.
