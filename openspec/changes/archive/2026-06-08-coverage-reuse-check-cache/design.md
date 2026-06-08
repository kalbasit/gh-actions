## Context

`ci.yml` exposes an opt-in `coverage` job that runs `nix build ".#<primary_package>.coverage" -L`
and uploads the result to Codecov. As written today (the `coverage:` job in
`.github/workflows/ci.yml`), it has **no `needs:`** on any other job, so GitHub schedules it
immediately and it runs concurrently with the check fan-out (`check-matrix` → `checks`),
`flake-check`, and `build`.

For a typical Nix consumer, the `.#<pkg>.coverage` derivation depends on the same test/cohort
derivations that the check jobs build. Those jobs push their build outputs to Cachix (via the
`cachix-action` post-build hook) as they run. When coverage runs *alongside* them, none of those
outputs are in Cachix yet, so the coverage build is a full cache miss and re-runs the entire
instrumented test suite itself.

Concrete evidence (the `ncps` consumer):
- Before `ncps` PR #1363, `ncps` carried its own `coverage` job with `needs: checks`; it built
  `.#ncps.coverage` *after* the cohort derivations were cached, hitting Cachix and finishing in
  **~33s** (a tiny Python merge of five `cover.out` profiles).
- PR #1363 moved coverage onto this shared job, which has no `needs`. Coverage now rebuilds the
  cohorts from scratch — the job log shows `ncps-test-cache> buildPhase completed in 2m33s` and
  `ncps-redis-tests> checkPhase completed in 1m40s` — and takes **~6m20s** every run since.

This is a pure scheduling/cache-timing regression: the coverage derivation is unchanged; only
*when* it runs relative to the cache-populating jobs changed.

## Goals / Non-Goals

**Goals:**
- Make the shared `coverage` job reuse the check jobs' Cachix-pushed artifacts instead of
  rebuilding them, restoring near-instant coverage for consumers whose coverage derivation shares
  dependencies with their checks.
- Keep the fix generic to the reusable workflow — correct for every `check_mode` (`single`,
  `matrix`, `none`) and for `oci: true`/`false` — with no consumer-side change.
- Preserve all existing coverage gating: the `inputs.coverage` toggle and the fork-PR /
  missing-`codecov_token` skip, and keep the final `ci` gate's treatment of the result intact.

**Non-Goals:**
- Changing the coverage derivation, the Codecov upload step, or any input/secret contract.
- Optimising the `check_mode: none` case — with no check jobs there is nothing to populate the
  cache, so coverage legitimately builds its dependencies itself. That is the consumer's choice.
- Restructuring coverage into per-check artifact uploads + a merge job (consumer-specific; the
  merge logic lives in the consumer's flake, not here).

## Decisions

### Decision 1: Serialize `coverage` after the check-producing jobs via `needs:`

Add `needs: [check-mode-guard, flake-check, check-matrix, checks, build]` to the `coverage` job.
GitHub will then start coverage only once all of those have reached a terminal state, by which
point their Cachix pushes are complete and coverage's shared dependencies are substitutable.

**Why these five jobs:** they are the complete set of cache-populating jobs in `ci.yml`. The
reusable workflow cannot know *which* specific derivations a given consumer's coverage build
depends on, so it waits for every job that could have built and pushed them. This is the same
approach the old `ncps` bespoke job took (`needs: checks`), generalised across modes.

**Alternative considered — depend only on `checks`:** matches `ncps` today but breaks other
modes. In `single` mode the cache is populated by `flake-check` (or, when `oci: true`, by the
inline check in `build.yml`), not `checks`. Depending only on `checks` would leave those
configurations rebuilding. Rejected in favour of waiting for all cache-populating jobs.

**Alternative considered — leave concurrent, rely on Cachix mid-run:** the post-build hook does
push incrementally, but coverage starts at t=0 and immediately demands all dependencies before
any check job has finished pushing. The race is unwinnable without ordering. Rejected.

### Decision 2: Gate with `!cancelled()` instead of implicit success-of-needs

A job with `needs:` runs by default only if **all** needed jobs succeed; if any needed job is
**skipped**, the dependent is skipped too. Several of the five dependencies are *always* skipped
in any given configuration:
- `flake-check` runs only when `oci == false && check_mode == single`.
- `check-matrix` / `checks` run only when `check_mode == matrix`.
- `build` runs only when `oci == true`.

So with a default `needs` gate, coverage would be cascade-skipped in essentially every real
configuration. The job's `if:` must therefore explicitly opt back in. Use:

```yaml
coverage:
  needs:
    - check-mode-guard
    - flake-check
    - check-matrix
    - checks
    - build
  if: >-
    ${{ !cancelled()
        && inputs.coverage
        && (github.event_name != 'pull_request'
            || github.event.pull_request.head.repo.fork == false) }}
```

`!cancelled()` evaluates true when dependencies are skipped or successful (and false only when the
run is cancelled), which is exactly the "run after they settle, tolerate skips" semantics we want.
The existing `inputs.coverage` and fork-PR conditions are preserved verbatim.

**Alternative considered — `always()`:** also tolerates skips, but additionally forces the job to
run even when the workflow is cancelled, wasting a runner and muddying the final gate. `!cancelled()`
is the tighter choice.

**On failed dependencies:** `!cancelled()` lets coverage run even if a check job *failed*. That is
acceptable and arguably desirable — coverage is independent signal, the final `ci` gate already
fails the run on the failed check, and a failed check means its outputs aren't cached so coverage
simply pays a cache miss (no worse than today). Encoding "skip coverage iff a relevant check
failed" would require enumerating per-mode result expressions for jobs that are themselves often
skipped — brittle for no real benefit. Explicitly out of scope.

### Decision 3: `ci` gate unchanged

The final `ci` job already lists `coverage` in its `needs` and treats `failure`/`cancelled` as a
gate failure while accepting `skipped`/`success`. Serializing coverage does not change which
results it can produce, so the gate needs no edit. The fork-PR skip scenario continues to pass the
gate.

## Risks / Trade-offs

- **[Coverage waits for the slowest check, adding wall-clock latency on the critical path]** →
  Net win: today coverage is itself the ~6m critical path (a full second suite run); after the
  fix the path is `slowest-check + ~30s merge`, which is shorter, and the duplicated compute is
  removed. For consumers whose coverage shares nothing with checks, latency is added with no cache
  benefit — but those consumers can simply leave `coverage: false` or accept the ordering; no
  consumer in this org is in that position.
- **[`!cancelled()` runs coverage after a failed check]** → Tolerable: the run already fails via
  the `ci` gate; coverage just pays a cache miss it would have paid anyway. Mitigation: documented
  as intentional in the spec ("runs when dependencies are skipped or successful").
- **[A future new cache-populating job is added to `ci.yml` but not to coverage's `needs`]** →
  coverage could again miss its cache. Mitigation: the spec's "Coverage upload" requirement names
  the dependency set explicitly so the linkage is reviewable; add new cache jobs to the list.

## Migration Plan

1. Edit only the `coverage:` job in `.github/workflows/ci.yml` (add `needs:`, replace `if:`).
2. Merge to `main`. Consumers pinned to `@main` (ncps, swm, signal-api-receiver) pick it up on
   their next CI run with no change on their side.
3. Validate on `ncps`: the next `coverage` job should drop back to ~30s with Cachix hits for the
   cohort derivations rather than rebuilding them.
4. Rollback: revert the single-job edit; behaviour returns to concurrent (slow) coverage. No state
   or contract migration involved.

## Open Questions

- None. The dependency set is the full enumeration of cache-populating jobs in `ci.yml`, and the
  `!cancelled()` gating is the standard idiom for "run after mode-dependent, often-skipped
  dependencies".
