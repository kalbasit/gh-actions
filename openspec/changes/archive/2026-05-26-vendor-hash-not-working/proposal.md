## Why

The built-in `go_deps` paths-filter watches only root-level `go.mod`/`go.sum`. Repos
with multiple Go modules (e.g. swm, where each plugin lives in its own subdirectory
module) miss filter hits when a subdirectory `go.mod` changes — so `generate` is
skipped, the `vendorHash` stays stale, and `flake-check` fails. Confirmed broken by
kalbasit/swm#158 (Renovate bumped `go-github` v67→v88 in `plugins/forge-github/`).

## What Changes

- **Widen the built-in `go_deps` filter** from `["go.mod", "go.sum"]` to
  `["**/go.mod", "**/go.sum"]` so that any Go module file at any depth triggers
  `generate`, not just the repo root.
- **Update the `dependency-hash-generation` spec** to document the corrected filter paths.

No breaking changes — consumers do not configure the built-in filter directly.

## Capabilities

### Modified Capabilities

- **`dependency-hash-generation`** — built-in `go_deps` filter paths change from
  `go.mod`/`go.sum` (root-only) to `**/go.mod`/`**/go.sum` (all depths).
  Delta spec file: `openspec/specs/dependency-hash-generation/spec.md`

## Impact

- `.github/workflows/ci.yml` — builtin filter config in the `filter` job's
  "Build paths-filter config" step
- `.github/workflows/generate.yml` — same builtin block (kept in sync)
- `openspec/specs/dependency-hash-generation/spec.md` — filter paths table updated
