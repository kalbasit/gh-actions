## Why

`ncps` hand-rolls a flake-check fan-out (`check-matrix` → `checks` matrix →
`coverage` jobs) in its own `ci.yml`, made necessary by the `run_flake_check: false`
escape hatch (change #9). That pattern — enumerate `.#checks.<system>` and build each
check on its own runner to collapse wall-clock — is generic and belongs in the reusable
workflow so every large consumer gets it without copy-paste, while quick repos keep the
simple single-job check.

## What Changes

- **BREAKING**: Remove the `run_flake_check` boolean from `ci.yml` and `build.yml`.
  Replace it with a `check_mode` enum input on `ci.yml`:
  - `single` (default) — one monolithic `nix flake check -L` per system (today's
    standalone `flake-check` job when `oci: false`, or the inline check inside the
    `build` matrix when `oci: true`). Good for quick repos.
  - `matrix` — enumerate `.#checks.<system>` and fan each check out into its own
    parallel job. Good for large projects.
  - `none` — skip inline checks entirely (replaces `run_flake_check: false`); OCI
    image build/push unaffected.
- Add the **matrix** fan-out to `ci.yml`: a `check-matrix` job enumerates check
  attribute names via `nix eval .#checks.<system> --apply builtins.attrNames --json`,
  feeding a `checks` job whose matrix is the **system × check** product. Systems come
  from `test_systems` (or all `systems` when empty). Each check runs
  `nix build ".#checks.$SYSTEM.$CHECK" -L`, passing matrix values through env vars (not
  `${{ }}` interpolation) to stay safe against template injection from fork PRs.
- **BREAKING**: Add a new opt-in `coverage` boolean input (default `false`). When
  `true`, a fork-safe `coverage` job builds `.#<primary_package>.coverage` and uploads
  to Codecov (scoped by `test_systems`, skipped on fork PRs / when no `codecov_token`).
  Because it defaults off, `oci: true` consumers that previously got inline coverage for
  free must now opt in.
- Extend the final `ci` gate to enumerate the new `check-matrix`, `checks`, and
  `coverage` jobs.
- Migrate `ncps/ci.yml`: delete its hand-rolled `check-matrix` / `checks` / `coverage`
  jobs, set `check_mode: matrix` and `coverage: true` on the shared call, and drop
  `run_flake_check: false`.

## Capabilities

### New Capabilities
<!-- None — this extends the existing reusable-CI contract rather than introducing a new capability. -->

### Modified Capabilities

- `reusable-ci`: Replace the `run_flake_check` input with the `check_mode` enum;
  add the `matrix` fan-out (`check-matrix` + `checks` jobs over the system × check
  product, scoped by `test_systems`, injection-safe); add the opt-in `coverage`
  input and its fork-safe Codecov job; update the inputs table, the final-gate
  enumeration, and fork-safety scenarios.
- `oci-build`: Replace `run_flake_check` forwarding in `build.yml` with `check_mode`
  handling — the inline monolithic `nix flake check` runs only in `single` mode;
  decouple the inline coverage steps from checks and gate them on the new `coverage`
  input instead of `run_flake_check`.

## Impact

- **Workflows**: `.github/workflows/ci.yml` (new inputs, matrix + coverage jobs, gate),
  `.github/workflows/build.yml` (input swap, coverage gating).
- **Specs**: `openspec/specs/reusable-ci/spec.md`, `openspec/specs/oci-build/spec.md`.
- **Consumers**: `ncps/ci.yml` migrates to `check_mode: matrix` + `coverage: true` and
  loses ~60 lines of bespoke fan-out. `ncps` is the only current consumer of
  `run_flake_check`, so removing it is safe to do in lockstep here. Quick consumers
  using defaults are unaffected (`check_mode` defaults to `single`).
- **Documentation**: `README.md` input/secret contract table.
