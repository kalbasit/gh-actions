## Context

The `ci.yml` reusable workflow has a `filter` job that uses `dorny/paths-filter` to
decide whether to run `generate`. The built-in `go_deps` filter pattern is
`["go.mod", "go.sum"]`. `dorny/paths-filter` treats patterns without a leading `**/`
as root-anchored globs, so only the repository root's `go.mod`/`go.sum` triggers the
filter. Repos with multiple Go modules (swm has a plugin per subdirectory, each with
its own `go.mod`) silently miss filter hits when a subdirectory module changes.

Confirmed broken: kalbasit/swm#158 (Renovate bumped `plugins/forge-github/go.mod`
from `go-github` v67→v88, root `go.mod` was untouched, `generate` was skipped,
`flake-check` failed with a stale `vendorHash`).

The same builtin block appears in both `ci.yml` (filter job) and `generate.yml`
(generate job) and must be kept in sync.

## Goals / Non-Goals

**Goals:**
- `generate` triggers whenever any `go.mod`/`go.sum` at any depth changes
- No consumer-side changes required

**Non-Goals:**
- Scoping filter to only the subdirectories referenced by `languages` targets (over-
  engineering; false positives from unrelated subdirectories are harmless — generate
  re-confirms hashes are up to date and no-ops if nothing changed)
- Fixing `go mod tidy` to handle multi-module repos (Renovate already produces correct
  `go.sum`; `go mod tidy` is a safety step for the root module only and doesn't block
  hash computation for subdirectory targets)

## Decisions

### Use `**/go.mod` and `**/go.sum` instead of root-anchored patterns

**Decision**: Change the builtin filter from `["go.mod", "go.sum"]` to
`["**/go.mod", "**/go.sum"]`.

**Rationale**: `**/` is the standard glob prefix for "any depth" in
`dorny/paths-filter`. This is a one-character change with no consumer impact. The
alternative — requiring consumers to declare extra filter paths per subdirectory
module via `extra_filters` — is worse ergonomics and would require Renovate PRs to
be manually patched each time a new plugin module is added.

**Alternatives considered**:
- Derive watched paths from the `languages` target file paths (e.g., infer
  `plugins/forge-github/go.mod` from `nix/packages/swm-plugin-forge-github/default.nix`).
  Rejected: the mapping from Nix package file to Go module root is not deterministic
  and would require new input fields.
- Document `extra_filters` as the workaround. Rejected: Renovate PRs are auto-merged;
  there is no human in the loop to add filters before the PR lands.

## Risks / Trade-offs

- **[Risk] Over-triggering `generate`** if a repo has non-language `go.mod`-named files
  in unexpected paths. → Mitigation: harmless — `generate` will run, confirm all hashes
  are current, and emit no commit.
- **[Risk] dorny/paths-filter `**` semantics differ between versions.** → Mitigation:
  the action is SHA-pinned; `**` behaviour is documented and stable across v3/v4.
