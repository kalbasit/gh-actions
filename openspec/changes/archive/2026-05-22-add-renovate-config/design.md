## Context

The proposal locks in *what* we want: Renovate watches the SHA-pinned actions in this
repo's workflows, opens grouped PRs that bump SHA + `# vN.N.N` comment in lockstep, and
auto-merges trusted-namespace digest bumps on green CI. This design covers the *how*:
where the config file lives, which Renovate features express each policy, and how the
trust boundary is enforced.

This repo is small (4 reusable workflows + 1 sanity-test workflow), uses the GitHub App
flavor of Renovate, and has no other dependency managers in play today. The design is
deliberately compact — there isn't much to invent, but there are a few decisions worth
recording.

## Goals / Non-Goals

**Goals:**

- One config file. No regex managers, no custom shell hooks, no `renovate-bot` workflow.
- The trailing `# vN.N.N` comment next to each SHA stays in sync automatically.
- Digest bumps from trusted upstreams auto-merge on green CI.
- Major-version bumps stay manual.
- Adding new managers later (flake.lock, docker-compose, …) is a one-line config edit.

**Non-Goals:**

- Self-hosting Renovate.
- Configuring Renovate in consumer repos (covered by a future change if they ever move
  to SHA pins).
- Renovating non-Actions dependencies in this repo (there are none yet).
- Replacing the GitHub App with `renovatebot/github-action` (Action-driven Renovate
  is heavier than the App for a small repo).

## Decisions

### D1: Config location — `.github/renovate.json5`

Renovate accepts any of `renovate.json`, `renovate.json5`, `.github/renovate.json`,
`.github/renovate.json5`, `.renovaterc`, etc. Choose **`.github/renovate.json5`**:

- `.github/` keeps repo-root tidy; aligns with `.github/workflows/`.
- `.json5` allows comments — useful for explaining the trust list and the auto-merge
  rationale to future maintainers without an out-of-band doc.

**Alternatives considered:** `renovate.json` at repo root (slightly more discoverable
but uncommented), `.github/renovate.json` (no comments). Comments win.

### D2: Base preset — `config:recommended`

Extend `config:recommended` (Renovate's curated baseline). It enables sensible defaults:
dependency dashboard issue, prHourlyLimit, ignoring unstable channels, etc. We override
only what we need.

**Why not `config:base`:** deprecated; `config:recommended` is its successor.

### D3: Single grouped PR per week for Actions digests

```json5
"packageRules": [
  {
    "matchManagers": ["github-actions"],
    "matchUpdateTypes": ["digest", "pinDigest"],
    "groupName": "github actions digests",
    "schedule": ["before 6am on Monday"],
    "automerge": true,
    "matchPackageNames": [
      "/^actions\\//",
      "/^cachix\\//",
      "/^codecov\\//",
      "/^docker\\//",
      "/^dorny\\//",
      "/^stefanzweifel\\//",
      "/^DeterminateSystems\\//"
    ]
  },
  {
    "matchManagers": ["github-actions"],
    "matchUpdateTypes": ["major"],
    "automerge": false
  }
]
```

(`matchPackagePatterns` is deprecated in current Renovate; the implementation uses
`matchPackageNames` with the regex-delimited form `/<pattern>/`, escaping the literal
`/` inside the pattern with `\/`.)

- One PR/week keeps noise low. Renovate still respects `prHourlyLimit` if multiple
  bumps queue up.
- The trust list is the union of every namespace currently pinned in the workflows.
  Adding a new upstream namespace requires editing this allowlist — friction is the
  point.
- Major bumps (`v6 → v7`) are excluded from auto-merge regardless of namespace; those
  imply breaking changes worth reading the changelog for.

**Alternative:** auto-merge all Actions digest bumps without a namespace allowlist. Too
broad — a careless `uses: third-party/random@<sha>` slipped in later would inherit
auto-merge implicitly. Explicit allowlist is safer.

### D4: How the `# vN.N.N` comment stays in sync

Renovate's built-in `github-actions` manager recognizes the pattern:

```yaml
uses: actions/checkout@<sha>   # v6
```

and updates both the SHA and the trailing version comment together. No custom regex
manager is needed. The comment must match the version pattern Renovate emits (it
writes `# v6`, `# v6.1.0`, etc. depending on `pinDigests`'s output style).

Verify with `renovate-config-validator` (one-time, in tasks).

### D5: Auto-merge gating

Renovate's `automerge` setting only fires when:

1. All required status checks pass.
2. The branch is up to date with `main`.

This repo's `_test-openspec-guard.yml` is a `workflow_dispatch` workflow with no
pull_request trigger, so currently the **only** check that runs against a Renovate PR
is whatever's wired in branch-protection. **For auto-merge to be meaningful, we need at
least one workflow that runs on `pull_request` and exercises the changed workflow file.**

Two options to satisfy this:

- **D5.a (chosen)**: add a tiny `.github/workflows/_test-workflows.yml` that runs
  `actionlint` (via `nix run nixpkgs#actionlint`) on every `pull_request` to `main`.
  Cheap (~30s), catches the most likely failure modes from an action upgrade
  (deprecated inputs, removed outputs).
- D5.b: trust Renovate without CI gating. Rejected — defeats the purpose.

Renovate also supports `platformAutomerge: true` (GitHub-native auto-merge instead of
Renovate doing the merge). Use it — leverages GitHub's branch-protection enforcement.

### D6: Branch protection compatibility

If/when branch protection lands on `main`:

- Require status checks (at minimum, the actionlint workflow from D5).
- Allow `renovate[bot]` to bypass the "pull request reviews" rule for digest bumps, OR
  require zero reviews (this repo is small enough that auto-merge with green CI is
  fine).
- Renovate's GitHub App needs `write` access to PRs and contents — granted by App
  install.

Document the required settings in tasks so future-me doesn't fight the platform.

### D7: Out-of-bounds versions / floating refs

Two pins are unusual:

- `DeterminateSystems/flake-checker-action@ba1df9ab06e525ea302f2a56f5e182f33cfd14fa # main`
  — the comment says `# main` because the upstream doesn't publish version tags. Pin
  to a SHA snapshotted from `main`. Renovate handles this via `pinDigests` (it'll
  track the branch tip and bump the SHA).
- All other actions are at version tags (`v6`, `v31`, etc.); Renovate's default
  semver-aware logic applies.

Tell Renovate the flake-checker pin is a branch-follow via a per-package rule with
`followTag` not needed — `pinDigest` from a branch is the default Renovate behavior.
Just make sure the comment Renovate updates next to the SHA is the branch name. (If
that's not how the github-actions manager renders branch-follow comments, the worst
case is the comment goes stale and we file an upstream issue or add a regex manager.)

## Risks / Trade-offs

- **Risk:** A compromised upstream tag → Renovate's digest bump points at the
  compromised commit → green CI (because actionlint doesn't detect malicious
  behavior, only schema issues) → auto-merge → silent rollout. **Mitigation:**
  namespace allowlist keeps the blast radius to vendors we trust; weekly cadence
  gives a ~7-day window where a published advisory could surface; in the worst case
  we revert.
- **Risk:** The actionlint workflow (D5.a) doesn't catch a runtime regression in an
  upstream action. **Mitigation:** the action's *own* CI is the first line of
  defense; we're the second. If the consumer repos (ncps, swm, sar) run on PRs that
  call ci.yml@main, *their* full CI doubles as our integration test for any action
  upgrade. (Slightly indirect but real.)
- **Trade-off:** Weekly grouping delays urgent security patches up to 7 days. We can
  add a `vulnerabilityAlerts` rule that bypasses the schedule for advisories tagged
  in GitHub's database — Renovate does this by default with `config:recommended`,
  so it's free.
- **Risk:** Adding a new pinned namespace and forgetting to extend the allowlist →
  Renovate opens digest bumps that don't auto-merge → backlog of stale PRs.
  **Mitigation:** Renovate's dependency dashboard issue surfaces unmerged PRs; weekly
  review of the dashboard is a low-cost habit.

## Migration Plan

1. Land the proposal/design/specs/tasks via this change.
2. Add `.github/renovate.json5` (per D1–D3).
3. Add `.github/workflows/_test-workflows.yml` (per D5.a) — runs actionlint on every
   PR.
4. User installs the Renovate App on `kalbasit/gh-actions` (one-time GitHub UI click).
5. Wait for Renovate's onboarding PR (auto-opened by the App). Merge it as-is — it
   contains nothing besides metadata.
6. Renovate opens its first weekly digest-bump PR Monday morning. Watch it land.

Rollback: delete the config file; the App's dependency dashboard will go quiet.
Uninstall the App via GitHub UI.

## Open Questions

- Should we ALSO enable `enabledManagers: ["github-actions"]` to be explicit about
  "Actions only" and avoid any future surprise when, say, a `package.json` lands in
  this repo? Probably yes — additive when we want more managers later.
- Should Renovate's PRs trigger the actionlint workflow even though branch protection
  isn't on yet? They will by default — `pull_request:` events fire regardless of
  protection rules. The protection layer is only relevant when *we* want to *require*
  the check.
