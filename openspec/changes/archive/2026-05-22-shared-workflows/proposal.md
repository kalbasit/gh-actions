## Why

Three of @kalbasit's Nix+Go repos (`ncps`, `swm`, `signal-api-receiver`) maintain near-identical
CI workflows that drift independently. The most recent (`swm`) gained an `openspec-guard` job
that the others lack; meanwhile fixes to vendor-hash logic, codecov upload, or multi-arch OCI image
must be ported by hand across repos. Centralizing the CI as reusable workflows in this repo
removes the drift, makes each consumer's `ci.yml` a thin caller, and lets new capabilities
(like the OpenSpec guard) light up everywhere via a single version bump.

## What Changes

- Add a reusable **`ci.yml`** workflow (`workflow_call`) that orchestrates the full pipeline:
  filter → generate vendor hashes → flake-check / build (with optional OCI image) → openspec-guard → final gate.
- Add a reusable **`build.yml`** workflow that performs `nix flake check` (per-arch),
  coverage upload to Codecov, multi-arch OCI image build/push, and multi-arch manifest
  creation. Triggered as a sub-workflow from `ci.yml` when OCI image support is enabled.
- Add a reusable **`generate.yml`** workflow that updates Nix dependency hashes for one or many
  packages and auto-commits the result. **Language-aware**: a `languages` input (JSON list,
  e.g. `["go"]`, `["python"]`, `["go","python"]`) selects which per-language generators run.
  Each language contributes its own paths-filter rule, its own list of package attrs to refresh,
  and its own "regenerate + rewrite hash in `default.nix`" recipe. The Go generator is the only
  one shipped initially (it implements today's `vendorHash` flow via `nix build .#<pkg>.goModules`),
  but the workflow is structured so adding Python (`buildPythonPackage` / poetry2nix /
  `cargoHash` for Rust / etc.) later is additive, not a rewrite.
- Add a reusable **`openspec-guard.yml`** workflow that fails CI when any active (non-archive)
  OpenSpec change exists under `openspec/changes/`. Sourced from SWM.
- All OCI-image-related behavior (build matrix, codecov coverage upload, multi-arch manifest)
  is gated by a single `oci` boolean input so `swm` (no images) can opt out while `ncps` /
  `signal-api-receiver` opt in.
- All repo-specific names (cachix cache name, Nix package names, OCI image names, docs
  project) are inputs rather than hardcoded.
- Language-agnostic steps (`nix flake check`, openspec-guard, OCI image build/push, multi-arch
  manifest, codecov upload, final gate) stay language-agnostic. Only the **generate** phase
  branches on `languages`; everything downstream depends solely on the resulting tree state.
- Update `openspec/config.yaml` in this repo with the project context (reusable GitHub Actions
  workflows for Nix+Go projects) and a few per-artifact rules to keep proposals tight.

**Scope includes migrating each consumer repo**: as part of this change, `../swm`,
`../ncps`, and `../signal-api-receiver` each get their `.github/workflows/ci.yml` rewritten
into a thin caller of `kalbasit/gh-actions/.github/workflows/ci.yml@<tag>`. ncps and
signal-api-receiver also lose their per-repo `build.yml`. Other workflows in those repos
(`releases.yml`, `flake-update.yml`, `devskim.yml`, etc.) are out of scope.

### Key differences between the three source workflows (captured for the implementer)

| Aspect | ncps | swm | signal-api-receiver |
| --- | --- | --- | --- |
| Cachix cache name | `ncps` | `swm` | `signal-api-receiver` |
| Vendor-hash packages | single (`ncps`) | five (`swm`, `swm-plugin-forge-github`, `swm-plugin-picker-fzf`, `swm-plugin-session-tmux`, `swm-plugin-vcs-git`) | single (`signal-api-receiver`) |
| OCI image build (multi-arch) | yes | **no** | yes |
| Codecov coverage upload | yes (`codecov-action@v6`) | no | yes (`codecov-action@v5`) |
| Database codegen step (`sqlc generate` + `go generate`) | yes | no | no |
| Cloudflare Pages docs deploy | yes | no | no |
| OpenSpec guard job | no | **yes (newest)** | no |
| `workflow_dispatch` trigger | yes | yes | **no** |
| Filter paths | `go.mod`, `go.sum`, `db/**`, `docs/**` | `go.mod`, `go.sum` | `go.mod`, `go.sum` |
| Final gate style | iterate `toJSON(needs)` | enumerate `needs.*.result` | iterate `toJSON(needs)` |
| codecov-action version | `@v6` | n/a | `@v5` |

The `nix build .#<pkg>.goModules` + sed-replace `vendorHash` recipe is byte-identical across all three
once parameterized by package name. The `nix flake check`, OCI image login, metadata, push, and multi-arch
manifest steps in `ncps/build.yml` and `signal-api-receiver/build.yml` are identical apart from the
cachix cache name and codecov-action version.

### Conflict-resolution precedence

When the three source workflows disagree on **how** something common is done, the newer repo
wins: **SWM > ncps > signal-api-receiver**. Concrete applications of this rule:

- **Vendor-hash generation**: adopt SWM's package-loop form even for single-package repos (the
  loop degenerates correctly with a one-element list). Commit message format follows SWM:
  `"chore: update vendor hashes"` (lowercase, conventional-commits prefix), not ncps's
  `"Auto Update Vendor Hash for <pkg>"`.
- **Final CI gate**: adopt SWM's explicit `needs.*.result` enumeration (clearer failure
  messages) over the `toJSON(needs) | grep` approach used by ncps and signal-api-receiver.
- **`workflow_dispatch` trigger**: included (SWM and ncps have it; signal-api-receiver's
  omission is the outlier).
- **Flake check**: `nix flake check -L` runs on every supported system. When `oci: false`
  (swm today), it runs in a standalone job per-system. When `oci: true`, it runs inside the
  per-arch OCI build matrix (as ncps/signal-api-receiver do today) so we don't double-pay for
  the same evaluation. Either way: x86_64-linux **and** aarch64-linux are exercised.
- **codecov-action version**: ncps's `@v6` wins over signal-api-receiver's `@v5` (SWM has no
  opinion — no coverage).
- **OpenSpec guard**: SWM-only, no conflict — included unconditionally.

Database codegen (ncps) and Cloudflare Pages docs deploy (ncps) are **out of scope** for this initial
change — they're ncps-only and can stay in that repo's `ci.yml` as an additional job that runs
alongside the reusable workflow, or be folded in later as additional opt-in inputs.

## Capabilities

### New Capabilities

- `reusable-ci`: end-to-end reusable CI workflow consumers call via `workflow_call`; orchestrates
  filtering, generation, build, openspec-guard, and final gating with optional OCI image support.
- `dependency-hash-generation`: reusable, language-aware job that recomputes Nix dependency
  hashes for one or more packages per enabled language and auto-commits. Ships with Go support
  (today's `vendorHash` flow); designed so Python / Rust / Node generators can be added later
  as additional opt-in entries in the `languages` input without restructuring the workflow.
- `oci-build`: reusable job that runs `nix flake check` per-arch, builds coverage, uploads to
  Codecov, and builds/pushes multi-arch OCI images plus a multi-arch manifest.
- `openspec-guard`: reusable job that fails when active (non-archived) OpenSpec changes exist.

### Modified Capabilities

(none — this repo currently has no specs)

## Impact

- **This repo**: new `.github/workflows/{ci,build,generate,openspec-guard}.yml` reusable
  workflows; updated `openspec/config.yaml`.
- **Consumer repos** (ncps, swm, signal-api-receiver): each gets its `ci.yml` rewritten as a
  thin caller; `build.yml` is deleted from ncps and signal-api-receiver. ncps additionally
  retains its `deploy-docs-pages` and sqlc database-codegen jobs as sibling jobs in its local
  `ci.yml` (still ncps-specific). All three consumer migrations are in scope for this change.
- **Secrets contract**: consumers must continue to pass `CACHIX_AUTH_TOKEN`, `GHA_PAT_TOKEN`,
  `CODECOV_TOKEN` (OCI builds only), `DOCKERHUB_TOKEN` (OCI builds only) — same secrets they
  already set. (Despite the name, `DOCKERHUB_TOKEN` is just the registry credential for the
  OCI registry on hub.docker.com; we push to both Docker Hub and ghcr.io.)
- **Versioning**: consumers pin via `@main` or a tag/SHA. Recommend tagging once stable.
