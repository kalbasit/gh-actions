## 1. Rewrite `generate.yml`

- [x] 1.1 Replace the Go module's `GO_PACKAGES_JSON` env + `for pkg in packages`
      loop with a `GO_TARGETS_JSON` env + jq-driven `while read entry` loop. Each
      iteration extracts `attr` and `file` and runs the existing verify-then-rebuild
      recipe, with the sed-replace targeting `$file` instead of the convention path.
- [x] 1.2 Extend the "Validate languages input" preflight to enforce: every enabled
      language has a non-empty `targets` array; each entry has exactly the keys
      `attr` and `file`; both values are non-empty strings. Implementation: a few
      `jq` expressions in the existing validation step.
- [x] 1.3 actionlint clean against `generate.yml`.

## 2. README

- [x] 2.1 Rewrite the `### languages shape` subsection: replace the
      `{"go":{"packages":[...]}}` JSON example with the `targets` shape. Document
      `attr` (Nix attribute path, language module appends a suffix like
      `.goModules`) and `file` (path of the file containing the `vendorHash`
      line).
- [x] 2.2 Update the swm consumer example: explicit `targets` array, one entry per
      package (5 entries).
- [x] 2.3 Update the ncps consumer example: two targets (`ncps` and
      `ent-codegen-drift-check`).
- [x] 2.4 Update the signal-api-receiver consumer example: single-entry `targets`
      array.

## 3. swm migration (`../swm`)

- [x] 3.1 Repin `kalbasit/gh-actions/.github/workflows/ci.yml@main` →
      `@feat-generate-configurable-targets`.
- [x] 3.2 Convert `languages: {"go":{"packages":[...]}}` to the explicit `targets`
      form (5 entries).
- [x] 3.3 actionlint + nix fmt + commit via /git-commit.

## 4. ncps migration (`../ncps`)

- [x] 4.1 Repin `kalbasit/gh-actions/.github/workflows/ci.yml@main` →
      `@feat-generate-configurable-targets` in `ci.yml`.
- [x] 4.2 Repin `build.yml@main` → `@feat-generate-configurable-targets` in
      `releases.yml`.
- [x] 4.3 Convert `ci.yml`'s `languages` to the `targets` form with TWO entries:
      `{attr: ncps, file: nix/packages/ncps/default.nix}` and
      `{attr: checks.x86_64-linux.ent-codegen-drift-check, file: nix/checks/flake-module.nix}`.
- [x] 4.4 Delete the local `Update vendor hash for ncps` and `Update vendor hash for
      ent-codegen-drift-check` steps from `generate-database`, plus the
      `stefanzweifel/git-auto-commit-action@v7` step gated on `go_deps`. Keep the
      `Regenerate Ent code`, `Format Code`, and Ent-gated auto-commit step — those
      aren't vendor hashes.
- [x] 4.5 actionlint + nix fmt + commit via /git-commit.

## 5. signal-api-receiver migration (`../signal-api-receiver`)

- [x] 5.1 Repin `kalbasit/gh-actions/.github/workflows/ci.yml@main` →
      `@feat-generate-configurable-targets` in `ci.yml`.
- [x] 5.2 Repin `build.yml@main` → `@feat-generate-configurable-targets` in
      `releases.yml`.
- [x] 5.3 Convert `ci.yml`'s `languages` to the explicit `targets` form (1 entry).
- [x] 5.4 actionlint + commit via /git-commit (nix fmt may skip due to the
      worktree+`.git/config` upstream issue we already documented).

## 6. End-to-end verification (user-driven; gated on push)

- [ ] 6.1 Push all four branches. Open / update PRs.
- [ ] 6.2 Verify swm PR CI is green (no Go-module change → `generate` skips
      cleanly).
- [ ] 6.3 Verify ncps PR CI is green:
      - `shared / generate` handles `ncps.goModules` AND
        `checks.x86_64-linux.ent-codegen-drift-check.goModules` if `go_deps` filter
        matches (force a synthetic `go.mod` bump on a scratch branch to exercise
        this if necessary).
      - `generate-database` no longer pushes a vendor-hash commit.
      - `openspec-guard` still fires (active `migrate-to-ent-and-atlas`); resolved
        per ncps's own workflow.
- [ ] 6.4 Verify signal-api-receiver PR CI is green.

## 7. Land + repin back to main

- [ ] 7.1 Merge gh-actions PR.
- [ ] 7.2 Repin each consumer's `ci.yml` / `releases.yml` from
      `@feat-generate-configurable-targets` → `@main`. One commit per consumer.
- [ ] 7.3 Merge each consumer PR.
- [ ] 7.4 Archive this openspec change.
