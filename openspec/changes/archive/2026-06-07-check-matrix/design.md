## Context

`ci.yml` currently runs flake checks one of three ways, selected by the `run_flake_check`
boolean (change #9):

- `oci: false`, `run_flake_check: true` → a standalone `flake-check` job, one monolithic
  `nix flake check -L` per system.
- `oci: true`, `run_flake_check: true` → the same check inline inside `build.yml`'s
  per-arch matrix, plus inline `.#<pkg>.coverage` + Codecov upload.
- `run_flake_check: false` → both skipped, so the consumer can fan checks out itself.

`ncps` is the only consumer that sets `run_flake_check: false`, and it pays for that
escape hatch by hand-rolling ~60 lines: a `check-matrix` job that enumerates
`.#checks.x86_64-linux` and a `checks` matrix job that builds each check on its own
runner, plus a standalone `coverage` job. That fan-out is generic. This change moves it
into the reusable workflow behind a `check_mode` selector and turns coverage into an
independent opt-in, so large consumers get the fan-out for free and quick consumers keep
the single-job check.

Only `gh-actions` (self-test) and `ncps` consume these workflows today, and `ncps` pins
`@main`, so there is no published tag to coordinate — the contract can change in lockstep.

## Goals / Non-Goals

**Goals:**

- Replace `run_flake_check` with a `check_mode` enum (`single` | `matrix` | `none`) that
  selects how `nix flake check` runs.
- Implement `matrix` mode in the orchestrator: a `system × check` fan-out scoped by
  `test_systems`, injection-safe against fork PRs.
- Make coverage an independent, opt-in capability (`coverage: true`, default `false`),
  decoupled from check execution and from `oci`.
- Delete `ncps`'s bespoke `check-matrix` / `checks` / `coverage` jobs by migrating it to
  the shared inputs.

**Non-Goals:**

- Changing the OCI image build/push or multi-arch manifest behaviour.
- Per-language check awareness — checks are discovered generically from
  `.#checks.<system>`; the `languages` input is untouched.
- Coverage merging across multiple systems / Codecov flags — coverage runs once (see
  Decisions).
- Publishing a version tag or back-compat alias for `run_flake_check`.

## Decisions

### D1: `check_mode` enum replaces `run_flake_check`

`ci.yml` gains `check_mode` (string, default `single`); `run_flake_check` is removed from
both `ci.yml` and `build.yml`. Mapping from old behaviour: `run_flake_check: true` →
`single`, `run_flake_check: false` → `none`; `matrix` is new.

- **Why enum over boolean(s)**: three mutually-exclusive behaviours read poorly as two
  booleans (`run_flake_check` + `flake_check_matrix` has an illegal `false`+`true`
  combination). The user framed this as "mode selection."
- **Why no alias**: `ncps` is the sole `run_flake_check` user and is updated in this same
  effort; carrying a deprecated alias is dead weight. An invalid `check_mode` value should
  fail fast.
- **Validation**: a guard step (or the job `if:` conditions) treats any value other than
  `single` / `matrix` / `none` as an error.

### D2: The matrix fan-out lives in `ci.yml`, not `build.yml`

For both `oci: false` and `oci: true`, `matrix` mode runs two orchestrator jobs:

- `check-matrix`: checkout + Nix + Cachix, then for each effective system compute
  `nix eval .#checks.<system> --apply builtins.attrNames --json` and emit a single
  `include` output — a JSON array of `{system, check}` objects.
- `checks`: `strategy.matrix.include: fromJson(needs.check-matrix.outputs.include)`,
  `fail-fast: false`, `runs-on` derived from `system` (`aarch64-linux` → `ubuntu-24.04-arm`,
  else `ubuntu-24.04`), running `nix build ".#checks.$SYSTEM.$CHECK" -L`.

When `oci: true` + `matrix`, `build.yml` builds/pushes the OCI image per arch but runs **no**
inline check; the checks fan out in `ci.yml` alongside it. When `oci: true` + `single`,
`build.yml` keeps its existing inline `nix flake check`.

- **Why the orchestrator, not the build workflow**: matrix mode must work for `oci: false`
  consumers too, and keeping a check job inside the OCI build matrix would entangle two
  axes (arch × check). Placing it in `ci.yml` keeps OCI a pure per-arch image build.
- **Alternative considered — one enumeration job per system** (rejected): produces N
  small jobs and an awkward downstream join; a single job emitting a combined
  `{system, check}` include list is simpler and matches how `matrix.include` is consumed.

### D3: Effective systems = `test_systems` else `systems`

The enumeration computes its system list as `test_systems` when non-empty, otherwise the
full `systems` array (mirroring the existing "empty `test_systems` means all systems" rule
already applied to single-mode checks and coverage).

- **Why**: consistency with the shipped `test_systems` semantics; a consumer that already
  scopes its suite to `x86_64-linux` gets a matrix only over that system.

### D4: Injection safety via env vars

Check names come from `builtins.attrNames` over a possibly attacker-controlled flake on
fork PRs. The `checks` job passes `matrix.system` / `matrix.check` through `env:` and
references `"$SYSTEM"` / `"$CHECK"` inside `run:` (quoted), never interpolating
`${{ matrix.* }}` into the script — the pattern `ncps` already uses.

### D5: Coverage is an independent opt-in job in `ci.yml`

A new `coverage` boolean input (default `false`) adds a single `coverage` job to `ci.yml`:
build `.#<primary_package>.coverage` and upload `result-coverage` to Codecov. It is
fork-safe (skipped on fork PRs / when `codecov_token` is absent) and requires
`primary_package`. Coverage is removed entirely from `build.yml`.

- **Why default off**: not all consumers produce coverage; the user asked for explicit
  opt-in. Consequence: `oci: true` consumers that previously got inline coverage must now
  set `coverage: true` (a BREAKING change, acceptable given the single lockstep consumer).
- **Why a single job, not per-system**: coverage is informational, not a gate, and
  `ncps`'s current standalone coverage job already runs once on `ubuntu-24.04`. Per-system
  coverage + Codecov merging is out of scope; the job builds on the primary system
  (first of `test_systems`, else `x86_64-linux`).
- **Why decoupled from `check_mode`**: coverage and check execution are orthogonal — a
  consumer may want `matrix` checks without coverage, or coverage without a fanned-out
  matrix.

### D6: Final gate enumerates the new jobs

The terminal `ci` gate adds `check-matrix`, `checks`, and `coverage` to its explicit
`needs` + per-job `result` enumeration (no `toJSON(needs)` iteration), treating `skipped`
as success so unused modes don't fail the gate.

## Risks / Trade-offs

- **Removing `run_flake_check` breaks `ncps` until it is migrated** → ship the `ncps`
  `ci.yml` migration as a companion PR; since `ncps` pins `@main`, land them close
  together. Rollback = revert both.
- **Coverage default-off silently drops coverage for any OCI consumer not updated** →
  only `ncps` is affected and it is migrated here (`coverage: true`); README documents the
  new required opt-in.
- **`check-matrix` enumeration adds one extra job (Nix cold-start) before checks run** →
  the job pulls the shared compile cache from Cachix, so it is cheap; net wall-clock still
  collapses versus one monolithic check on a contended runner.
- **An empty `.#checks.<system>` yields an empty matrix** → `checks` simply produces zero
  jobs; the gate treats the skipped/empty result as success. Document that `matrix` mode on
  a flake with no checks is a no-op.
- **Invalid `check_mode` value** → fail fast in a guard rather than silently defaulting, so
  typos surface immediately.

## Migration Plan

1. In `gh-actions`: add `check_mode` + `coverage` inputs to `ci.yml`, implement the
   `check-matrix` / `checks` / `coverage` jobs, swap `build.yml` from `run_flake_check` to
   `check_mode` (inline check only in `single`) and remove its coverage steps.
2. Update the `reusable-ci` and `oci-build` specs and the `README.md` contract table.
3. Companion PR in `ncps`: delete its `check-matrix` / `checks` / `coverage` jobs and the
   `run_flake_check: false` input; set `check_mode: matrix` and `coverage: true` on the
   shared call; trim the final-gate `needs`.
4. Verify via this repo's `_test-*` harness / a `ncps` PR run that both `single` (other
   consumers' default) and `matrix` paths are green.
- **Rollback**: revert the `gh-actions` change and the `ncps` companion PR together.

## Open Questions

- Should `check_mode: matrix` also drive the `oci: true` inline path to *always* fan out,
  or is leaving `single` as the only inline-in-build mode (current decision) acceptable
  long-term? Resolved as the latter for now to bound the change.
- Coverage on the primary system only is assumed adequate; revisit if a consumer needs
  arch-specific coverage.
