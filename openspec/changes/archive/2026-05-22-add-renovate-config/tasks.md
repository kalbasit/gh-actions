## 1. Renovate config

- [x] 1.1 Add `.github/renovate.json5` extending `config:recommended` with
      `enabledManagers: ["github-actions"]`, the grouped-weekly-PR rule, the
      trusted-namespace auto-merge rule, the major-bump deny rule, and
      `platformAutomerge: true`. — Used the modern `matchPackageNames` regex form
      (`/^actions//`, …) instead of the deprecated `matchPackagePatterns`.
- [x] 1.2 Validate locally: `nix shell nixpkgs#nodejs --command npx --yes --package
      renovate -- renovate-config-validator .github/renovate.json5`. — "Config
      validated successfully".

## 2. CI gate

- [x] 2.1 Add `.github/workflows/_lint-workflows.yml`: triggered on `pull_request` to
      `main` (plus `workflow_dispatch`), runs `nix run nixpkgs#actionlint --
      .github/workflows/*.yml`. SHA-pinned `actions/checkout` and
      `cachix/install-nix-action`. Self-lint passed.
- [ ] 2.2 Verify the workflow runs and is green against the current branch's HEAD
      (push and observe). *(deferred until push)*

## 3. Documentation

- [x] 3.1 Update README.md to add a "Renovate" subsection pointing at
      `.github/renovate.json5` and explaining the weekly cadence + auto-merge
      allowlist.
- [x] 3.2 Note in the README that the Renovate GitHub App must be installed for the
      config to take effect (link to https://github.com/apps/renovate).

## 4. One-time install (user-driven)

- [ ] 4.1 Install the Renovate GitHub App on `kalbasit/gh-actions` via
      https://github.com/apps/renovate (user action, not automatable from inside the
      repo).
- [ ] 4.2 Merge the Renovate-opened onboarding PR as-is once it appears.

## 5. End-to-end verification

- [ ] 5.1 Confirm the dependency dashboard issue (auto-opened by Renovate) lists every
      pinned action from `build.yml`, `ci.yml`, `generate.yml`, `openspec-guard.yml`,
      and the new `_lint-workflows.yml`.
- [ ] 5.2 Force a test bump: temporarily un-pin one action (drop a few characters from
      its SHA), push the branch, and confirm Renovate opens a corrective PR. Revert
      the test bump.
- [ ] 5.3 Confirm the first natural weekly PR (Monday) auto-merges on green CI.
- [ ] 5.4 D7 edge case: on the first PR that bumps
      `DeterminateSystems/flake-checker-action`, confirm Renovate preserves the
      trailing `# main` comment (since this pin follows a branch, not a tag). If
      Renovate drops or rewrites the comment, file a follow-up to add a regex
      manager that keeps the literal `# main` in place.

## 6. Future-proofing notes (not implementation; document in design or PR description)

- [x] 6.1 Record in this change's archive notes: how to extend the namespace
      allowlist. — Documented inline in `.github/renovate.json5`'s comment header on
      the auto-merge rule: "adding a new namespace requires a deliberate edit here".
- [x] 6.2 Record: how to add a new manager later. — Documented inline in
      `.github/renovate.json5`'s comment on `enabledManagers`: "If a package.json or
      flake.lock lands later, append its manager to this list — that's the entire
      opt-in."
