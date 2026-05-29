# Tasks: skip-monolithic-flake-check

- [x] 1. `build.yml`: add `run_flake_check` input (bool, default `true`); AND it
  into the `if:` of the Flake check, Build coverage, and Codecov steps.
- [x] 2. `ci.yml`: add `run_flake_check` input; forward to `build.yml`; gate the
  standalone `flake-check` job (`oci == false && run_flake_check`).
- [x] 3. Spec deltas: `oci-build` (conditional check/coverage) + `reusable-ci`
  (new input).
- [x] 4. `actionlint` clean (exit 0).
- [ ] 5. `openspec validate`; commit; open PR.
