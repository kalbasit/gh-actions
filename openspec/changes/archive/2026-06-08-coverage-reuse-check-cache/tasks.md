## 1. Implement the coverage job ordering

- [x] 1.1 In `.github/workflows/ci.yml`, add `needs: [check-mode-guard, flake-check, check-matrix, checks, build]` to the `coverage:` job.
- [x] 1.2 Replace the `coverage:` job `if:` with a `!cancelled()`-gated condition that preserves the existing `inputs.coverage` toggle and the fork-PR / `head.repo.fork == false` skip (see design.md Decision 2).
- [x] 1.3 Confirm the `coverage` job's `runs-on` primary-system expression and the `Build coverage` / `Upload coverage reports to Codecov` steps are otherwise unchanged.

## 2. Verify gating across modes

- [x] 2.1 Confirm the `if:` expression is valid YAML/GitHub-expression syntax (`nix run nixpkgs#actionlint -- .github/workflows/*.yml` → exit 0).
- [x] 2.2 Reason through each configuration and confirm coverage is NOT cascade-skipped: `check_mode: matrix` (flake-check + build skipped), `check_mode: single` + `oci: false` (check-matrix/checks + build skipped), `check_mode: single` + `oci: true` (flake-check + check-matrix/checks skipped). `!cancelled()` leading the `if:` overrides the default skipped-need cascade in all three.
- [x] 2.3 Confirm the fork-PR case still skips coverage and the final `ci` gate still treats the skip as success (the `ci` job needs no edit). Also confirmed `push` to `main` and `workflow_dispatch` still run coverage (fork clause is true for non-`pull_request` events).

## 3. Spec + housekeeping

- [x] 3.1 Run `openspec validate --specs --no-interactive` and the repo's openspec-guard expectations; ensure the change validates. (change valid; specs 5/5 pass)
- [x] 3.2 Commit on a feature branch with a Conventional Commit message (e.g. `fix(ci): run coverage after check jobs to reuse cachix cache`).
- [ ] 3.3 After merge to `@main`, validate on the `ncps` consumer that the next `coverage` job hits Cachix and returns to ~30s instead of rebuilding the suite.
- [x] 3.4 Archive this change once implemented (openspec-guard requires no active change at merge).
