## Why

The reusable `generate.yml` workflow hard-codes a magic relationship between a Go
package name and the file that holds its `vendorHash`:

- Input: `languages.go.packages = ["ncps"]`
- Implicit Nix attr: `ncps.goModules`
- Implicit file: `nix/packages/ncps/default.nix`

This convention covers the typical case but breaks when a project has Go modules
inside attributes that don't live under `nix/packages/<name>/default.nix`. ncps's
`ent-codegen-drift-check` is exactly that case — its `vendorHash` lives in
`nix/checks/flake-module.nix` under the Nix attr
`checks.x86_64-linux.ent-codegen-drift-check.goModules`. ncps's caller works around
this today by re-implementing the verify-then-sed recipe locally in
`generate-database`, which:

1. **Races with `generate.yml`** for the `nix/packages/ncps/default.nix` update when
   both fire on the same `go.mod` change (two `git-auto-commit-action` pushes
   against the same file).
2. **Duplicates the verify-then-sed code path** — the same SWM-derived recipe is now
   maintained in two places.

Rather than paper over this with an additive `extra_targets`, take the chance to
remove the magic entirely. Every target gets named explicitly: where the Nix attr
lives, where the hash sed-replace runs. The convention is gone; the workflow does
exactly what the caller tells it to.

## What Changes

- **BREAKING**: replace `languages.go.packages: ["<name>"]` with
  `languages.go.targets`, a list of explicit `{ attr, file }` objects. Every
  consumer's caller must list each target explicitly.

  Old:
  ```json
  {"go": {"packages": ["ncps"]}}
  ```

  New:
  ```json
  {"go": {"targets": [
    {"attr": "ncps", "file": "nix/packages/ncps/default.nix"}
  ]}}
  ```

  ncps with its check derivation:
  ```json
  {"go": {"targets": [
    {"attr": "ncps", "file": "nix/packages/ncps/default.nix"},
    {"attr": "checks.x86_64-linux.ent-codegen-drift-check", "file": "nix/checks/flake-module.nix"}
  ]}}
  ```

  `attr` is the Nix attribute path the workflow appends `.goModules` to and feeds to
  `nix build`. `file` is the path of the file containing the `vendorHash = "...";`
  line to sed-replace.

- `generate.yml` is simplified: one loop over `targets`. No path interpolation, no
  convention. The validation step still rejects unknown language keys.

- **All three consumer callers** (`../swm`, `../ncps`, `../signal-api-receiver`)
  must be updated. swm gains 5 explicit entries (one per package); ncps gains 2
  (`ncps` + the check derivation); signal-api-receiver gains 1.

- **ncps caller cleanup**: remove the local vendor-hash blocks from
  `generate-database` and the corresponding `git-auto-commit-action` step
  (`generate.yml` now handles both `ncps` and `ent-codegen-drift-check` via
  `targets`). Keep the ent codegen steps (`nix develop --command go generate
  ./ent/...` and `nix fmt`) and their auto-commit — those aren't vendor hashes.

- README updated: `languages` shape subsection rewritten around `targets`; three
  consumer examples (swm/ncps/signal-api-receiver) updated to the new form.

- The `dependency-hash-generation` capability spec has its existing requirements
  MODIFIED to describe `targets` instead of `packages`; the existing scenarios are
  updated; a new scenario covers a non-convention file path.

## Capabilities

### New Capabilities

(none)

### Modified Capabilities

- `dependency-hash-generation`: input shape changes — `packages: [<string>]` becomes
  `targets: [{attr, file}]`. The "Language-aware dependency hash regeneration"
  requirement, the "Language extension contract" requirement, and their scenarios
  are rewritten to reflect the explicit shape. **This is a breaking spec change
  consumers must adopt in lockstep with the workflow.**

## Impact

- **gh-actions**: `.github/workflows/generate.yml`'s Go module is rewritten — one
  loop over `targets` instead of the per-`packages` loop with implicit path
  construction. `README.md` `languages` shape section + all three consumer examples
  updated. `openspec/specs/dependency-hash-generation/spec.md` MODIFIED.

- **swm caller** (`../swm/.github/workflows/ci.yml`): the multi-line `packages: [...]`
  list becomes a multi-line `targets: [...]` list with one entry per package, each
  with explicit `attr` and `file`. ~10 lines net add (one extra line per package).

- **ncps caller** (`../ncps/.github/workflows/ci.yml`): `packages: ["ncps"]`
  becomes `targets: [<ncps>, <ent-codegen-drift-check>]`. The local
  `generate-database` job loses its two vendor-hash blocks and their
  `git-auto-commit-action` step (~60 lines removed); the ent codegen + nix fmt +
  auto-commit step stays. Net deletion.

- **signal-api-receiver caller** (`../signal-api-receiver/.github/workflows/ci.yml`):
  `packages: ["signal-api-receiver"]` becomes a one-entry `targets` list. Tiny.

- **Backwards compatibility**: none. Callers on `@main` will all break the moment
  `generate.yml` lands; the three caller PRs must be merged before (or with) the
  gh-actions PR, or the gh-actions config must temporarily accept both shapes during
  the migration. Design will pick between "land everything atomically via the
  feat-branch repin pattern we used previously" vs "ship a one-release transitional
  fallback that accepts both shapes and emits a deprecation `::warning::`".
