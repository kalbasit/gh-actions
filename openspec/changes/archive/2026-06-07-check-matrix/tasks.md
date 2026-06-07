## 1. ci.yml inputs and mode guard

- [x] 1.1 Add `check_mode` input (string, default `single`) and `coverage` input (boolean, default `false`) to `ci.yml`'s `workflow_call.inputs`, with descriptions matching the spec
- [x] 1.2 Remove the `run_flake_check` input from `ci.yml`
- [x] 1.3 Add a guard (job or step) that fails fast when `check_mode` is not one of `single` / `matrix` / `none`

## 2. Single mode (preserve current behavior)

- [x] 2.1 Gate the existing standalone `flake-check` job on `inputs.oci == false && inputs.check_mode == 'single'` (replacing the `run_flake_check` condition), keeping the `test_systems` scoping on the check step
- [x] 2.2 Confirm the `flake-check` job no longer references `run_flake_check`

## 3. Matrix mode fan-out in ci.yml

- [x] 3.1 Add a `check-matrix` job (checkout + install-nix + cachix) gated on `inputs.check_mode == 'matrix'` that computes the effective systems (`test_systems` if non-empty, else `systems`), runs `nix eval .#checks.<system> --apply builtins.attrNames --json` per system, and emits a single `include` output: a JSON array of `{system, check}` objects
- [x] 3.2 Add a `checks` job (`needs: check-matrix`, gated on `matrix` mode) with `strategy.fail-fast: false` and `matrix.include: fromJson(needs.check-matrix.outputs.include)`, selecting `runs-on` per `matrix.system` (`ubuntu-24.04-arm` for `aarch64-linux`, else `ubuntu-24.04`)
- [x] 3.3 Implement the check step as `nix build ".#checks.$SYSTEM.$CHECK" -L`, passing `matrix.system`/`matrix.check` via `env:` (`SYSTEM`/`CHECK`) and never interpolating `${{ matrix.* }}` into `run:` (injection safety)

## 4. Coverage job in ci.yml

- [x] 4.1 Add a `coverage` job gated on `inputs.coverage == true`, fork-safe (skip on fork PRs / when `codecov_token` is absent), that builds `.#<primary_package>.coverage` on the primary system (first of `test_systems`, else `x86_64-linux`) and uploads `result-coverage` via `codecov-action`
- [x] 4.2 Document that `primary_package` is required when `coverage: true`

## 5. build.yml (oci-build) changes

- [x] 5.1 Add `check_mode` input to `build.yml`; remove `run_flake_check`
- [x] 5.2 Run the inline `nix flake check -L` only when `check_mode == 'single'` (keeping `test_systems` scoping); skip it for `matrix` and `none`
- [x] 5.3 Remove the inline coverage build + Codecov upload steps from `build.yml` (coverage now lives in `ci.yml`); drop the now-unused `codecov_token` secret if no longer referenced
- [x] 5.4 Update `ci.yml`'s `build` job invocation to forward `check_mode` instead of `run_flake_check`

## 6. Final gate

- [x] 6.1 Add `check-matrix`, `checks`, and `coverage` to the terminal `ci` gate's `needs` and its explicit per-job `result` enumeration, treating `skipped` as success

## 7. Documentation

- [x] 7.1 Update `README.md` input/secret contract table: replace `run_flake_check` with `check_mode`, add `coverage`, and document the `matrix` mode behavior and `primary_package`/`codecov_token` requirements
- [x] 7.2 Add a short consumer example for `check_mode: matrix` + `coverage: true`

## 8. Self-test harness

- [x] 8.1 Review `_test-*` / `_lint-workflows.yml` harness and add or update coverage for the new `check_mode` paths if the harness exercises ci.yml — no harness exercises `ci.yml` directly (`_lint-workflows.yml` runs actionlint over all workflows, which now covers the new jobs); nothing to add
- [x] 8.2 Run `actionlint` / the repo's workflow lint locally and fix any findings — `nix run nixpkgs#actionlint -- .github/workflows/*.yml` passes (exit 0)

## 9. ncps migration (sibling repo at ../ncps)

- [x] 9.1 In `../ncps/.github/workflows/ci.yml`, delete the bespoke `check-matrix`, `checks`, and `coverage` jobs
- [x] 9.2 On the `shared` call, remove `run_flake_check: false` and add `check_mode: matrix` and `coverage: true`
- [x] 9.3 Remove `check-matrix`/`checks`/`coverage` from the ncps final-gate `needs` and result enumeration
- [x] 9.4 Confirm the ncps `languages.go.targets` entry for the codegen-drift check is still valid and that no remaining job references the deleted ones — verified; actionlint on the ncps workflow passes (exit 0)

## 10. Verification and archive

- [x] 10.1 Validate the OpenSpec change: `openspec validate check-matrix` (passes)
- [x] 10.2 Verify `single` mode (default) and `none` mode still behave as before; verify `matrix` mode fans out per system × check on a real ncps PR run (or the self-test harness) — confirmed green on ncps PR #1363 (pinned to gh-actions #12)
- [x] 10.3 Confirm coverage uploads on a non-fork ncps build and is skipped on fork PRs — confirmed on ncps PR #1363
- [x] 10.4 Archive the change once implemented and merged (`openspec archive check-matrix`), coordinating the ncps companion PR — specs synced and change archived; ncps repin tracked on PR #1363
