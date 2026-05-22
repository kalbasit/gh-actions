# renovate-config Specification

## Purpose
TBD - created by archiving change add-renovate-config. Update Purpose after archive.
## Requirements
### Requirement: Renovate configuration file

The repository SHALL ship a Renovate configuration at `.github/renovate.json5` that
extends `config:recommended` and explicitly enables only the `github-actions` manager.

#### Scenario: Renovate App installed

- **WHEN** the Renovate GitHub App is installed on the repository
- **THEN** Renovate SHALL read `.github/renovate.json5` and open its onboarding PR
  reflecting the configured policies (single grouped weekly PR for Actions digest bumps,
  manager scoped to `github-actions`)

#### Scenario: Config validity

- **WHEN** `npx --yes --package renovate -- renovate-config-validator
  .github/renovate.json5` is run
- **THEN** the validator SHALL exit 0 with no errors

### Requirement: Trusted-namespace auto-merge allowlist

The configuration SHALL auto-merge digest-only updates **only** for packages whose names
start with one of the following namespaces:

- `actions/`
- `cachix/`
- `codecov/`
- `docker/`
- `dorny/`
- `stefanzweifel/`
- `DeterminateSystems/`

Updates outside this allowlist SHALL be opened as PRs that require human review.

#### Scenario: Trusted-namespace digest bump

- **WHEN** Renovate detects that `actions/checkout@<oldSha>` (where the pin's tag, e.g.
  `v6`, now resolves to `<newSha>`) is out of date
- **THEN** Renovate SHALL open a PR bumping `<oldSha>` to `<newSha>` (and updating the
  `# v6.x.y` trailing comment) with `automerge: true`
- **AND** when CI on that PR is green, the PR SHALL merge without human intervention

#### Scenario: Untrusted-namespace digest bump

- **WHEN** a workflow later references an action outside the allowlist (e.g.,
  `random-vendor/foo`) and that pin needs a digest bump
- **THEN** Renovate SHALL open a PR that does NOT auto-merge
- **AND** the PR SHALL remain open until a human reviewer merges it

### Requirement: Major-version bumps require human review

For any package matched by the `github-actions` manager, Renovate SHALL open
major-version updates (e.g. v6 to v7) with `automerge: false`, and such PRs SHALL
require explicit human review and merge regardless of namespace.

#### Scenario: Major-version bump in a trusted namespace

- **WHEN** `actions/checkout` releases `v7` and Renovate proposes a major upgrade
- **THEN** Renovate SHALL open a PR with `automerge: false`
- **AND** the PR SHALL require human review and explicit merge

### Requirement: Weekly grouped PR cadence

All `github-actions` digest bumps SHALL be grouped into a single PR per week,
scheduled `before 6am on Monday` (per Renovate's timezone defaults). Vulnerability
alerts (`config:recommended`'s `vulnerabilityAlerts` behavior) SHALL bypass this
schedule.

#### Scenario: Multiple digest bumps in the same week

- **WHEN** `actions/checkout`, `cachix/install-nix-action`, and `docker/login-action`
  all have new SHAs in the same week
- **THEN** Renovate SHALL open a SINGLE PR titled "github actions digests" containing
  all three bumps, scheduled for the next Monday before 6am

#### Scenario: GitHub-published vulnerability advisory

- **WHEN** GitHub publishes a security advisory affecting a pinned action
- **THEN** Renovate SHALL open a remediation PR immediately, bypassing the weekly
  schedule

### Requirement: Trailing version comment must stay in sync

Renovate SHALL update the trailing `# vN.N.N` comment in lockstep with the SHA on every
pinned action so that human readers can see the resolved version without manually
decoding the SHA. The expected pin form is `uses: <owner>/<repo>@<full-sha>   # vN.N.N`.

#### Scenario: SHA bump preserves comment format

- **WHEN** Renovate bumps a pinned action's SHA from `<oldSha>` (commented `# v6.0.0`)
  to `<newSha>` (which is the tip of `v6.1.0`)
- **THEN** the resulting PR diff SHALL update both the SHA and the comment to
  `# v6.1.0`

### Requirement: CI gates auto-merge

The repository SHALL run a `pull_request`-triggered workflow that exercises the changed
workflow files (at minimum, lints them with `actionlint`). Renovate SHALL be configured
with `platformAutomerge: true` so that GitHub's branch-protection / required-check
machinery is the gating layer for auto-merge.

#### Scenario: Auto-merge blocked by failing CI

- **WHEN** Renovate opens a digest-bump PR but the lint workflow fails on it
- **THEN** the PR SHALL NOT merge (GitHub's auto-merge requirement fails)
- **AND** the PR SHALL remain open with a visible failed-check status

#### Scenario: Auto-merge proceeds on green CI

- **WHEN** Renovate opens a digest-bump PR and all required status checks are green
- **THEN** GitHub-native auto-merge SHALL merge the PR without further human action

