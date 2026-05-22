## 1. Repo setup

- [x] 1.1 Update `openspec/config.yaml` with project context (reusable GitHub Actions workflows
      for Nix-flake Go projects; conflict precedence SWM > ncps > signal-api-receiver) and
      per-artifact rules (keep proposals under 800 words; tasks atomic; specs require ≥1
      scenario per requirement).
- [x] 1.2 Add a top-level `README.md` documenting the four reusable workflows, their input
      shapes, secret contracts, and a minimal "consumer thin caller" example.
- [x] 1.3 Decide on the Cachix cache name(s) used in local examples (use whichever consumer
      repo is being migrated first). — Decision: `your-cache` placeholder in the quick-start
      example, plus dedicated swm/ncps/signal-api-receiver consumer examples below it.

## 2. `openspec-guard.yml`

- [x] 2.1 Implement `.github/workflows/openspec-guard.yml` as a `workflow_call` workflow with
      a single job that checks out the consumer repo, scans `inputs.path` (default
      `openspec/changes`) for non-archive subdirectories, writes a job summary, and fails
      with the SWM-style error if any are found.
- [x] 2.2 Add `path` input (default `openspec/changes`).
- [ ] 2.3 Sanity-test by calling it from a scratch workflow in this repo with both a fake
      "active" directory and a clean state. *(deferred — requires push to actually run on
      GitHub Actions; covered as part of §6 end-to-end testing)*

## 3. `generate.yml`

- [x] 3.1 Implement `.github/workflows/generate.yml` (`workflow_call`) accepting `cachix_cache`,
      `languages` (JSON object), and `extra_filters` (JSON object).
- [x] 3.2 Implement the built-in `dorny/paths-filter` config (Go paths today; structure
      ready for python/rust additions).
- [x] 3.3 Implement the Go language module: `go mod tidy` → for each pkg in
      `languages.go.packages`, run SWM's two-build verify-then-rewrite recipe; emit
      `::notice::Hash is up to date for <pkg>` on no-change, sed-replace `vendorHash =
      "...";` in `nix/packages/<pkg>/default.nix` on mismatch.
- [x] 3.4 Add the validation step that fails with `::error::Unknown language: <key>` for any
      unrecognized key in `languages`.
- [x] 3.5 Add `stefanzweifel/git-auto-commit-action@v7` step at the end (one commit per run,
      message `chore: update vendor hashes`).
- [x] 3.6 Add the fork-PR skip guard mirroring SWM's `if:` condition.

## 4. `build.yml` (OCI)

- [x] 4.1 Implement `.github/workflows/build.yml` (`workflow_call`) with inputs `systems`,
      `images`, `dockerhub_username`, `push_oci`, `cachix_cache` and secrets
      `cachix_auth_token`, `codecov_token`, `dockerhub_token`.
- [x] 4.2 Implement the per-arch matrix selecting `ubuntu-24.04-arm` for `aarch64-linux`,
      `ubuntu-24.04` otherwise.
- [x] 4.3 Implement: checkout → flake-checker → install-nix → cachix → version.txt write (on
      semver ref) → `nix flake check -L` → `nix build .#<primary>.coverage` (skipped on fork)
      → `codecov-action@v6` upload (skipped on fork) → `nix build
      .#packages.<system>.docker` → push-docker-image build (if `push_oci`) → Docker Hub +
      ghcr.io login (if `push_oci` && not fork) → `docker/metadata-action@v6` → push.
- [x] 4.4 Add the follow-up `oci-manifest` job (depends on the matrix job) that builds and
      pushes the multi-arch manifest, using SWM/ncps's loop over emitted tags.
- [x] 4.5 Determine how to derive the "primary package" name for `version.txt` and the
      coverage attr (likely a new `primary_package` input; document the assumption).
      — Decision: explicit required input `primary_package` on `build.yml` and (when `oci:
      true`) on `ci.yml`. Documented in README.

## 5. `ci.yml` orchestrator

- [x] 5.1 Implement `.github/workflows/ci.yml` (`workflow_call`) with all inputs from the
      reusable-ci spec.
- [x] 5.2 Implement the `filter` job that always runs, emitting per-language outputs.
- [x] 5.3 Wire `generate` to call `generate.yml` (using `uses:` workflow chaining or inline
      the steps — decide based on whether we want one job or a nested workflow file).
      — Chose nested workflow_call via `uses: ./.github/workflows/generate.yml`.
- [x] 5.4 Wire `flake-check` job (matrix over `systems`) — runs only when `oci: false`.
- [x] 5.5 Wire `build` job calling `build.yml` — runs only when `oci: true`.
- [x] 5.6 Wire `openspec-guard` (calling `openspec-guard.yml`, unconditional).
- [x] 5.7 Implement the final `ci` job with `if: always()` and SWM's enumerated
      `needs.*.result` checks; print job-specific error messages.
- [x] 5.8 Wire all required secrets through (`cachix_auth_token`, `gha_pat_token`,
      `codecov_token`, `dockerhub_token`).

## 6. End-to-end testing in this repo

- [x] 6.1 ~~Create a `.github/workflows/_test-ci.yml` that calls `./.github/workflows/ci.yml`
      with a fake `languages` payload, against a checkout of one consumer's flake.~~
      — Scaled back: there is no flake in this repo, so a true end-to-end test of
      `generate` / `flake-check` / `build` is not honest here. Added
      `.github/workflows/_test-openspec-guard.yml` (workflow_dispatch) which calls
      `openspec-guard.yml` against this repo's own `openspec/changes`. The real
      end-to-end test of the orchestrator happens during the swm migration (§7).
- [ ] 6.2 Verify the matrix runs on both `ubuntu-24.04` and `ubuntu-24.04-arm`.
      *(verified during swm migration once the orchestrator runs there)*
- [ ] 6.3 Tag `v0.1.0` once the test workflow goes green at least once.

## 7. Migrate `../swm`

- [x] 7.1 In `../swm`, replace `.github/workflows/ci.yml` with a thin caller of
      `kalbasit/gh-actions/.github/workflows/ci.yml@feat-add-reusable-ci-workflows` setting
      `cachix_cache: swm`, `oci: false`, and the five Go packages. — Committed as
      `1963b74` on `user/wnasreddine/ci-jobs`. Will repin to `@v0.1.0` when tagged.
- [ ] 7.2 Open the PR, confirm CI is green, confirm OpenSpec guard still fires (test by
      adding then removing an active change), confirm generate still works on a synthetic
      `go.mod` bump.
- [ ] 7.3 Merge.

## 8. Migrate `../ncps`

- [x] 8.1 In `../ncps`, replace `.github/workflows/ci.yml` with a thin caller setting
      `cachix_cache: ncps`, `oci: true`, `images: kalbasit/ncps`,
      `dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}`,
      `languages: '{"go":{"packages":["ncps"]}}'`. — Committed as `115fca5`.
- [x] 8.2 Keep `deploy-docs-pages` and the sqlc/`go generate` "Generate Database Code" step
      as **sibling jobs in ncps's local `ci.yml`**. — Sibling jobs run their own
      `dorny/paths-filter` since the reusable workflow doesn't (today) re-expose filter
      outputs. A local final-gate job wraps both the shared workflow and the siblings.
- [x] 8.3 Delete `../ncps/.github/workflows/build.yml` (replaced by reusable `build.yml`).
- [ ] 8.4 Open the PR, confirm CI is green on both arches, confirm codecov upload still
      happens, confirm docs deploy still happens, confirm OCI images publish to Docker Hub
      and ghcr.io with the correct multi-arch manifest.
- [ ] 8.5 Merge.

## 9. Migrate `../signal-api-receiver`

- [x] 9.1 In `../signal-api-receiver`, replace `.github/workflows/ci.yml` with a thin caller
      setting `cachix_cache: signal-api-receiver`, `oci: true`,
      `images: kalbasit/signal-api-receiver`,
      `dockerhub_username: ${{ vars.DOCKERHUB_USERNAME }}`,
      `languages: '{"go":{"packages":["signal-api-receiver"]}}'`. — Committed as `552c1e4`.
- [x] 9.2 Add `workflow_dispatch:` to the caller. — Done.
- [x] 9.3 Delete `../signal-api-receiver/.github/workflows/build.yml`.
- [ ] 9.4 Confirm codecov upload now uses `@v6` (was `@v5`) and that coverage still appears
      in the codecov UI.
- [ ] 9.5 Open the PR, confirm CI is green on both arches, merge.

## 10. Finalize

- [ ] 10.1 Once all three consumers have been merged and green for at least one week, cut
      `v1.0.0` of this repo.
- [ ] 10.2 Open follow-up PRs in each consumer bumping the pin from `@v0.1.0` to `@v1`.
- [ ] 10.3 Archive this OpenSpec change.
