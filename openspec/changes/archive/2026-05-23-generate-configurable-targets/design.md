## Context

The proposal pins the *what*: replace `languages.go.packages: ["<name>"]` (which
implicitly resolves attr=`<name>.goModules` and file=`nix/packages/<name>/default.nix`)
with `languages.go.targets: [{attr, file}]` (where both fields are explicit). This is
a breaking input-shape change to `kalbasit/gh-actions/.github/workflows/generate.yml`
and the orchestrator chain that calls it. The motivation is the ncps duplication issue
where one target lives at `nix/checks/flake-module.nix` and can't be expressed by the
old convention, plus the more general win of removing magic from a workflow that other
people will configure.

The user owns all three consumer repos, so a breaking change is cheap. This design
locks the migration strategy, the input validation, and the loop shape inside
`generate.yml`.

## Goals / Non-Goals

**Goals:**

- Single explicit per-target shape `{attr, file}` for every language.
- One loop in `generate.yml` per language; no path interpolation, no convention.
- All three consumer callers updated atomically with the gh-actions change so no
  consumer is ever pointed at a `generate.yml` that doesn't understand its input.
- ncps's local vendor-hash blocks deleted; gh-actions handles both `ncps` and
  `ent-codegen-drift-check` via the new `targets` field.
- Spec gets a MODIFIED-style update, not an ADDED one — the existing requirements
  about hashed packages are rewritten, not extended.

**Non-Goals:**

- A transitional shim that accepts both `packages` and `targets`. Adds engineering
  overhead for a one-time, one-week migration we control end-to-end.
- Renaming `languages.go` to anything else. The per-language map keyed by language
  identifier is fine; only the per-language config object changes.
- Adding `extra_targets` or any other field. There's just `targets`.
- Validating that `attr` actually resolves at config-parse time. The existing
  build-and-capture-`got:` recipe already surfaces typos as a build failure with a
  legible Nix error.

## Decisions

### D1: Input shape — flat `targets` list, no convention

Per language:

```json
{
  "go": {
    "targets": [
      { "attr": "ncps", "file": "nix/packages/ncps/default.nix" },
      { "attr": "checks.x86_64-linux.ent-codegen-drift-check", "file": "nix/checks/flake-module.nix" }
    ]
  }
}
```

`attr` is the bare Nix attribute path (the language module appends the
hash-derivation suffix). `file` is the path containing the `vendorHash = "...";` line
to sed-replace. Both fields are required; both are validated by the workflow's
preflight step (D3).

**Alternatives considered:**

- *Keep `packages` AND add `extra_targets`*: leaves the magic in place for the easy
  case but means two code paths in `generate.yml` and two ways to express the same
  thing. User explicitly rejected this.
- *`targets` as a map keyed by attr*: `{ "ncps": "nix/packages/ncps/default.nix" }`.
  Slightly less verbose but rules out multiple targets sharing an attr (none today,
  but the asymmetry between key and value is ugly).
- *Array of `[attr, file]` 2-tuples*: shorter but less self-documenting. JSON objects
  with named keys win on readability.

### D2: `generate.yml` loop shape (Go module)

Replace today's:

```yaml
- name: Update Go vendor hashes
  if: fromJson(inputs.languages).go != null && steps.filter.outputs.go_deps == 'true'
  env:
    GO_PACKAGES_JSON: ${{ toJson(fromJson(inputs.languages).go.packages) }}
  run: |
    ...
    for pkg in "${packages[@]}"; do
      # build .#$pkg.goModules
      # sed into nix/packages/$pkg/default.nix
    done
```

with:

```yaml
- name: Update Go vendor hashes
  if: fromJson(inputs.languages).go != null && steps.filter.outputs.go_deps == 'true'
  env:
    GO_TARGETS_JSON: ${{ toJson(fromJson(inputs.languages).go.targets) }}
  run: |
    ...
    # jq -c iterates: each entry is the JSON object {"attr":..., "file":...}
    while IFS= read -r entry; do
      attr="$(echo "$entry" | jq -r .attr)"
      file="$(echo "$entry" | jq -r .file)"
      # ... build .#${attr}.goModules + sed the new hash into "$file"
    done < <(echo "$GO_TARGETS_JSON" | jq -c '.[]')
```

The verify-then-rebuild recipe inside the loop is unchanged from today — only the
iteration variable and the file path source move.

### D3: Preflight validation

The existing "Validate languages input" step gains target validation:

- Every enabled language's `targets` MUST be a non-empty array (consumers don't enable
  a language without targets).
- Each target MUST be an object with **exactly** the keys `attr` and `file`, both
  non-empty strings.
- File paths are NOT checked for existence at this step (the build/sed step will fail
  legibly when a path is wrong).

Failure modes:

- `::error::languages.go.targets must be a non-empty array`
- `::error::languages.go.targets[N] is missing required key 'attr'` (or `'file'`)
- `::error::languages.go.targets[N] contains unknown key '<key>'`

Implementation: pure `jq` in the validation step. No external deps.

### D4: Migration strategy — atomic via temporary feat-branch repin

The same pattern we used during the initial workflow rollout:

1. Land the gh-actions change on a feature branch (this branch).
2. On each consumer repo, open a PR that simultaneously:
   - Repins `uses: kalbasit/gh-actions/.github/workflows/ci.yml@<feat-branch>` (and
     `build.yml@<feat-branch>` for releases.yml).
   - Updates the caller's `languages.go.targets` to the new shape.
   - (ncps only) Deletes the local vendor-hash blocks from `generate-database`.
3. Verify each consumer's PR is green against the feat branch.
4. Merge the gh-actions PR to main.
5. Repin each consumer back to `@main` and merge.

**Rationale:** the user controls all three consumers and previously executed this
exact sequence cleanly. A transitional shim that accepts both shapes inside
`generate.yml` would mean ~30 lines of dead code shipping to `main` and being removed
in a follow-up — needless churn.

**Rollback:** if the feat branch breaks something we can't fix quickly, revert each
consumer's caller PR (restores the old `packages` shape pointing at `@main`, which
still has the old workflow). The gh-actions feat branch can sit unused without harm.

### D5: README + consumer-example updates

- Rewrite the `languages` shape section: drop the `packages` JSON example, replace
  with a `targets` example. Add a one-sentence note clarifying `attr` (Nix
  attribute path, will receive a language-specific suffix like `.goModules`) and
  `file` (file containing the hash line).
- Update all three consumer examples (swm/ncps/signal-api-receiver) to the new shape.
- ncps's example gains the `ent-codegen-drift-check` target as a worked example of
  "this is what a non-convention target looks like."

### D6: Spec delta shape — MODIFIED, not ADDED

The existing two requirements that reference `packages` ("Language-aware dependency
hash regeneration" and "Language extension contract") get full-replacement MODIFIED
treatment in the delta spec, including all their existing scenarios rewritten plus
one new scenario (D1's `ent-codegen-drift-check` worked example). The "Built-in
paths-filter rules", "Single commit per run", and "Fork PRs are skipped" requirements
are unchanged — the input shape doesn't touch them.

## Risks / Trade-offs

- **Trade-off:** swm's caller goes from `packages: ["swm", "swm-plugin-forge-github",
  "swm-plugin-picker-fzf", "swm-plugin-session-tmux", "swm-plugin-vcs-git"]`
  (5 strings) to a 5-element `targets` array with `{attr,file}` objects (~15 lines
  with one entry per line vs ~7 today). More typing for the simple case in exchange
  for never having to remember the convention. The user has explicitly accepted this.
- **Risk:** an atomic migration means a brief window (between merging gh-actions
  and merging each consumer's repin-back-to-main PR) where consumers are still
  pinned to the feat branch. If the feat branch is renamed or deleted prematurely,
  CI breaks. → **Mitigation:** mirror what we did last time: don't delete the feat
  branch until all three consumers are back on `@main` and green.
- **Risk:** a typo in `attr` doesn't get caught at validation time. → **Mitigation:**
  the existing `nix build` step fails legibly with the Nix error. The validation
  step's job is only to catch shape errors (missing keys, wrong types) where the
  failure mode would otherwise be confusing.
- **Risk:** during the migration, a Renovate digest-bump PR on gh-actions auto-merges
  in the middle of the rollout. → No risk: Renovate bumps don't touch
  `generate.yml`'s Go module, and consumer callers pinned to `@feat-...` don't see
  Renovate's `@main` changes anyway.

## Migration Plan

(See D4.) Concretely:

1. Implement gh-actions changes on the current feat branch
   (`feat-generate-configurable-targets`): `generate.yml` rewrite, README updates,
   delta spec, archive change.
2. Stack one branch per consumer (`user/wnasreddine/ci-jobs` continuation in each)
   that repins to the gh-actions feat branch and updates `targets`.
3. Open all four PRs. Verify CI is green on each.
4. Merge the gh-actions PR.
5. Repin consumers back to `@main` and merge.

## Open Questions

- Should `attr` be allowed to contain Nix-evaluation expressions (e.g. interpolations)
  in the future? Today, no — it's treated as an opaque string concatenated with
  `.goModules`. If a real use case emerges later, we can extend the validation. Not
  in scope here.
- Should the workflow also expose the resolved hash as a step output so the eventual
  commit log can reference both old and new SHAs? Maybe, but it's
  not motivated by ncps's use case. Defer.
