## Why

The reusable workflows in this repo now pin every third-party GitHub Action to a full
commit SHA (e.g. `actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6`). SHA
pinning hardens supply-chain security but only if the SHAs stay current — a stale pin
silently misses bug fixes and security patches. Without automation, those pins decay
because nobody remembers to look up the new SHA when a tag moves. Renovate solves this:
it watches each pinned action upstream and opens PRs that bump both the SHA and the
trailing `# vN.N.N` comment in lockstep, so the file stays honest.

## What Changes

- Add `.github/renovate.json` (or `renovate.json5`) configuring the Renovate GitHub App
  to track this repo's workflow files.
- Configure the `github-actions` manager with `pinDigests: true` so version-tag updates
  flow as digest bumps (matching how we pin today).
- Enable **auto-merge** for digest-only updates of well-known actions (`actions/*`,
  `cachix/*`, `docker/*`, `codecov/*`, `dorny/*`, `stefanzweifel/*`,
  `DeterminateSystems/*`). The whole point of SHA pinning is reproducibility; if a tag
  moves and the upstream is one we trust, we want the bump applied automatically with
  a green CI run as the gate. Auto-merge will only fire on a passing CI run, so
  `_test-openspec-guard.yml` and any future test workflow act as the safety net.
- Group all GitHub-Action digest updates into a single weekly PR (Mondays) so we don't
  drown in 9 separate PRs every time `actions/checkout`'s nightly tag moves.
- Keep major-version updates **non-automerged**: those imply API/behavior changes worth
  reading the changelog for.
- Update the README's "Versioning" / "Conventions" section to point at Renovate as the
  source of truth for action versions.
- Document a `renovate-bot` PR-author allowlist convention so future branch-protection
  changes know which automated commits to trust.

This change does **not**:

- Install the Renovate App on the repo — that's a GitHub UI/admin step the user does
  once. Tasks call it out as a manual step.
- Configure Renovate in the consumer repos (`ncps`, `swm`, `signal-api-receiver`).
  Those pin to **tag-refs** (`@main`, `@v0.1.0`) not SHAs, so Renovate gives them
  little value here; if/when those repos move to SHA pins, the same config can be
  copy-pasted.
- Configure Renovate for non-Actions managers (docker-compose, flake.lock, npm, etc.).
  This repo has none of those today; the config will use targeted managers so adding
  more later is a one-line change.

## Capabilities

### New Capabilities

- `renovate-config`: declarative configuration of the Renovate GitHub App for this repo,
  covering manager selection, digest pinning, grouping/scheduling, auto-merge policy,
  and the trust list of action namespaces eligible for auto-merge.

### Modified Capabilities

(none — Renovate is configuration only; the existing reusable-ci / oci-build /
dependency-hash-generation / openspec-guard capabilities are unchanged.)

## Impact

- **This repo**: new `renovate.json` (or `.json5`) at the repo root or under `.github/`.
  README updated. No workflow changes (Renovate runs as an App, not a workflow).
- **External**: one-time manual install of the Renovate GitHub App on
  `github.com/kalbasit/gh-actions` — captured as a task, not done by this change.
- **Branch protection**: if branch protection is later enabled on `main`, the rules
  must allow the `renovate[bot]` user to merge digest-bump PRs without a human review
  (or auto-merge won't actually merge). Captured in the design notes.
- **Trust boundary**: auto-merge means a compromised upstream tag → green CI →
  unattended deploy. Mitigations (allowlist of namespaces, digest-only bumps, CI as
  gate) are detailed in design.
