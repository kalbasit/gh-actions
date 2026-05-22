## Context

This repo will host reusable GitHub Actions workflows consumed cross-repo via
`uses: kalbasit/gh-actions/.github/workflows/<name>.yml@<ref>`. The initial consumers are
`ncps`, `swm`, and `signal-api-receiver` (Nix-flake Go projects, two of them with multi-arch
OCI images). The proposal establishes the *what* (one `ci.yml` orchestrator + `build.yml`,
`generate.yml`, `openspec-guard.yml` sub-workflows), the SWM > ncps > signal-api-receiver
precedence rule, the `oci` opt-in toggle, and the language-aware generate phase.

This design covers the *how*: input shapes, job graph, secret plumbing, and the per-language
extension contract for `generate.yml`.

## Goals / Non-Goals

**Goals:**

- One thin caller (`ci.yml` in the consumer repo) that names the cachix cache, lists packages,
  flips the `oci` switch, lists supported `systems`, and forwards secrets — nothing else.
- Language-aware generate phase that is **additive**: dropping in Python or Rust later is a
  new entry in a structured input, not a restructured workflow.
- Single source of truth for the SWM-style OpenSpec guard, the vendor-hash recipe, and the
  multi-arch OCI build/push/manifest dance.
- Final CI gate has clear, individually-named failure messages (SWM's enumerated style).
- Forks: jobs that need write tokens (`generate`, OCI push, codecov) must be skipped on
  PRs from forks without breaking the final gate.

**Non-Goals:**

- Database codegen (sqlc) and Cloudflare Pages docs deploy — ncps-only, stay in ncps's repo.
- Releases, backport, devskim, flake-update, semantic-pull-request — not in scope; they remain
  per-repo.
- Non-Nix projects. The workflow assumes a flake with `nix flake check` and per-package
  attributes; Nix is a hard requirement, not an input.
- (none — consumer-repo migrations were initially out of scope but are now bundled into this
  change; see the Migration Plan below.)

## Decisions

### D1: Structured object input for `languages`

Use a single JSON-encoded **object** input (not a flat list) keyed by language identifier,
with per-language configuration nested underneath:

```yaml
languages: |
  {
    "go": {
      "packages": ["swm", "swm-plugin-forge-github", "swm-plugin-picker-fzf"]
    }
  }
```

For a hypothetical future Python addition:

```yaml
languages: |
  {
    "go":     { "packages": ["myapp"] },
    "python": { "packages": ["myapp-py"], "lockfile": "poetry.lock" }
  }
```

**Why:** GitHub Actions `workflow_call` inputs are scalar-only (no native object/list types).
The structured form keeps related data together (per-language packages, future per-language
knobs like `lockfile`, `extra_paths_filter`) without polluting the input list with N inputs
per language. Consumers pay one JSON-string price; the workflow `fromJson`'s it once. The
flat alternative (`go_packages`, `python_packages`, …) couples input cardinality to language
count and forces the workflow to know every language by name upfront.

**Alternatives considered:**

- *Flat scalar inputs (`go_packages`, `python_packages`)*: simpler for one language, but adding
  a language requires editing the reusable workflow's input list, defeating "additive."
- *Matrix-driven (`fromJson(inputs.languages)` as a matrix)*: works for the generate step
  itself, but doesn't carry the auxiliary per-language config (lockfile path, extra filters).

### D2: Job graph

```
filter  ──► generate (per language, only on writeable PRs)
        ──► flake-check   (matrix: x86_64-linux, aarch64-linux)   [if oci=false]
        ──► build         (matrix: x86_64-linux, aarch64-linux)   [if oci=true]
        ──► openspec-guard
                            │
                            ▼
                          ci (final gate, if: always())
```

- `filter` runs `dorny/paths-filter` once and emits one output per known filter (`go_deps`,
  `python_deps`, …, plus a synthetic `any_deps` for the generate gate).
- `generate` is a single job that conditionally runs each language's regenerate-and-rewrite
  block based on `filter` outputs AND the `languages` input. Steps for absent languages are
  `if`-skipped, so a `["go"]`-only consumer sees zero Python steps.
- `flake-check` runs `nix flake check -L` once **per system** in the `systems` input (default
  `["x86_64-linux", "aarch64-linux"]`) on the matching runner (`ubuntu-24.04-arm` for
  `aarch64-linux`, `ubuntu-24.04` otherwise). It exists **only when `oci: false`**; when
  `oci: true` the `nix flake check` step lives inside the per-arch `build` matrix instead
  (matching today's ncps/signal-api-receiver layout), so we never pay for two flake
  evaluations per arch.
- `build` (gated by `oci: true`) runs the per-arch matrix: flake-check → coverage build →
  codecov upload → OCI image build → push (when not a fork) → multi-arch manifest in a
  follow-up `oci-manifest` job.
- `openspec-guard` runs unconditionally on a single runner.
- `ci` depends on all of the above with `if: always()`, enumerating each `needs.<job>.result`
  per SWM's pattern (clearer failure messages than `toJSON(needs) | grep`).

### D3: OCI opt-in is a single boolean

`oci: false` (default) skips the entire `build` job (no codecov, no OCI login, no
manifest). The final gate treats a skipped `build` as success (SWM's `result == "failure" ||
"cancelled"` check, not `!= "success"`). Codecov is intentionally bundled into `build` rather
than its own job — every consumer that wants coverage also wants OCI, and the coverage
artifact is produced by the same flake check that the OCI job already runs.

**Alternative:** split coverage out as `coverage: true|false`. Rejected as YAGNI given the
current consumer set; can be promoted to its own input later without breaking callers.

### D4: Generate phase: per-language contract

Each language module is a `composite-action`-shaped block inside `generate.yml`:

1. Read `fromJson(inputs.languages).<lang>` — if null, skip.
2. Read the matching `filter` output (`needs.filter.outputs.<lang>_deps`) — if false, skip.
3. Run the language-specific regenerate (Go: `go mod tidy`; Python (future):
   `poetry lock --no-update`; etc.).
4. For each package in `<lang>.packages`, run the language's "verify and rewrite hash" recipe:
   - **Go**: `nix build .#<pkg>.goModules` → on hash mismatch, `sed` the new hash into
     `nix/packages/<pkg>/default.nix`. Follow SWM's two-build verification (build, then
     rebuild) before declaring "up to date."
   - **Python (future)**: TBD; expected shape is the same — try to build, capture mismatch,
     rewrite.
5. After all languages run, a single `stefanzweifel/git-auto-commit-action@v7` step commits
   any changes with message `"chore: update vendor hashes"` (SWM's wording, conventional-
   commits prefix).

**Why one commit per run, not per language:** the consumer's history reads `chore: update
vendor hashes` regardless of whether one or three lockfiles moved. Reverting a generate run
is one commit.

### D5: Filter inputs default to the union of enabled languages

The consumer does **not** pass paths-filter rules. `generate.yml` (and `ci.yml` upstream of it)
ships a built-in `dorny/paths-filter` config that maps each known language to its dep paths:

- `go`: `go.mod`, `go.sum`
- `python` (future): `pyproject.toml`, `poetry.lock`, `requirements*.txt`
- `rust` (future): `Cargo.toml`, `Cargo.lock`

Consumers enabling a language automatically get its filter. An `extra_filters` input (JSON
object) allows ad-hoc additions (e.g., ncps's `db/migrations/**`) without forking the
workflow — but ncps's database codegen itself stays in ncps's repo per non-goals.

### D6: Fork safety

Steps that need write tokens (`GHA_PAT_TOKEN`, OCI push, codecov) check
`github.event.pull_request.head.repo.fork == false` as today. The `generate` job skips
entirely on forks (its `if:` mirrors the existing SWM/ncps gate). The final `ci` job treats
the resulting `skipped` as success.

### D7: Versioning of this repo

Consumers pin via tag (`@v1`) or SHA. We'll cut `v0.1.0` once `ncps`, `swm`, and
`signal-api-receiver` have each consumed the workflow at least once. Until then, `@main` is
acceptable. A floating `v1` tag is moved forward inside the major.

## Risks / Trade-offs

- **Risk:** Cross-repo workflow_call adds a network/permissions hop. Failures in this repo's
  workflows immediately break all three consumers. → **Mitigation:** consumers pin to a tag,
  not `@main`; this repo's `main` is protected; changes here open a PR in at least one
  consumer before tag bump.
- **Risk:** The `languages` JSON-object input is stringly-typed; a typo (`"goo"`) silently
  no-ops. → **Mitigation:** `generate.yml` validates known keys at the top of the job and
  fails with `::error::Unknown language: goo`. Add a list of accepted keys to the input
  description.
- **Trade-off:** One reusable `ci.yml` orchestrator means consumers can't easily reorder jobs
  or add custom jobs *between* (e.g., ncps's docs deploy between flake-check and build).
  → **Mitigation:** ncps's docs deploy stays a sibling job in ncps's local `ci.yml`,
  independent of the reusable call. The reusable workflow is one job graph; the local file
  can add more graph alongside it.
- **Risk:** Auto-commit on generate can race with the user's local commits. → **Mitigation:**
  unchanged from today — `git-auto-commit-action` already handles this via push + retry.
  Behavior is identical to current per-repo CI.
- **Trade-off:** Bundling codecov inside `build` couples coverage to OCI. → If a future
  consumer wants coverage without OCI, split via the YAGNI escape hatch in D3.

## Migration Plan

This change ships the reusable workflows **and** migrates all three consumer repos:

1. Land the reusable workflows in this repo → tag `v0.1.0`.
2. Open PR in `../swm` replacing `.github/workflows/ci.yml` with a thin caller; verify CI green.
3. Open PR in `../ncps` (keep docs-deploy and db-codegen as local sibling jobs); delete
   `../ncps/.github/workflows/build.yml`.
4. Open PR in `../signal-api-receiver`; delete its `build.yml`.
5. Once all three are green for at least a week, cut `v1.0.0` here and bump each consumer's
   pin from `@v0.1.0` to `@v1`.

Each consumer PR is independent. The order (swm → ncps → signal-api-receiver) is chosen so
the simplest consumer (no OCI, no codecov) exercises the orchestrator first; failures there
are cheaper to diagnose than failures in the multi-arch OCI path.

**Rollback:** each consumer migration is its own PR. Reverting that PR restores the
pre-existing per-repo workflow verbatim. This repo's workflows can sit unused without harm.

## Open Questions

- Should `openspec-guard` accept a `path` input (default `openspec/changes`) so non-default
  OpenSpec layouts work? Probably yes — cheap, additive.
- ~~Should `flake-check` run on the matrix systems (x86_64 + aarch64) or just x86_64?~~
  **Resolved:** all three source repos already exercise both arches (ncps and
  signal-api-receiver run `nix flake check -L` inside the per-arch build matrix; SWM does it
  in a standalone job — currently x86_64 only because SWM has no matrix, but consumers should
  get the same multi-arch coverage swm/aarch64 users rely on). The standalone `flake-check`
  job introduced here when `oci: false` runs the matrix over `systems` (default
  `["x86_64-linux", "aarch64-linux"]`).
- `codecov-action@v6` (ncps) vs `@v5` (signal-api-receiver): proposal locked this to `@v6`.
  Confirmed.
