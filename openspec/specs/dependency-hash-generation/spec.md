# dependency-hash-generation Specification

## Purpose
TBD - created by archiving change shared-workflows. Update Purpose after archive.
## Requirements

### Requirement: Language-aware dependency hash regeneration

The repository SHALL provide a reusable workflow at `.github/workflows/generate.yml` that,
for each language enabled in the structured `languages` input, regenerates the language's
dependency lock state, recomputes the Nix dependency hash for each explicit target
declared under that language, rewrites the matching `vendorHash = "...";` line in the
file the target names, and auto-commits the result.

Each target SHALL be an object of the form `{ "attr": "<nix-attribute-path>", "file":
"<path-to-default.nix-or-equivalent>" }`. The workflow appends `.goModules` (for Go) to
`attr` when invoking `nix build`. There is no convention path; the caller declares both
the attribute and the file explicitly.

#### Scenario: Go consumer with one convention-shaped target and out-of-date hash

- **WHEN** invoked with
  `languages: '{"go":{"targets":[{"attr":"myapp","file":"nix/packages/myapp/default.nix"}]}}'`
  and the `go.sum` of the consumer repo has changed
- **THEN** the workflow SHALL run `go mod tidy`, attempt `nix build .#myapp.goModules`,
  capture the `got:` hash from the build log on mismatch, replace the `vendorHash = "...";`
  line in `nix/packages/myapp/default.nix` with the new hash, and commit the change with
  message `chore: update vendor hashes`

#### Scenario: Go consumer with multiple convention-shaped targets, all up to date

- **WHEN** invoked with
  `languages: '{"go":{"targets":[{"attr":"a","file":"nix/packages/a/default.nix"},
  {"attr":"b","file":"nix/packages/b/default.nix"}]}}'` and all hashes match
- **THEN** for each target the workflow SHALL run `nix build .#<attr>.goModules` and
  then `nix build .#<attr>.goModules --rebuild` to confirm; SHALL emit `::notice::Hash
  is up to date for <attr>`; SHALL NOT create a commit

#### Scenario: Go consumer with a non-convention target file

- **WHEN** invoked with a target whose attribute lives outside `nix/packages/`, e.g.
  `{"attr":"checks.x86_64-linux.ent-codegen-drift-check","file":"nix/checks/flake-module.nix"}`,
  and that hash is out of date
- **THEN** the workflow SHALL run `nix build
  .#checks.x86_64-linux.ent-codegen-drift-check.goModules`, capture the new hash on
  mismatch, replace the `vendorHash = "...";` line in `nix/checks/flake-module.nix`
  with the new hash, and include the change in the same `chore: update vendor hashes`
  commit as any other target changes from the same run

#### Scenario: Unknown language key

- **WHEN** `languages` contains a key that is not a recognized language (e.g., `"goo"`)
- **THEN** the workflow SHALL fail with `::error::Unknown language: goo` before running any
  generator

#### Scenario: Empty language config

- **WHEN** `languages` is `{}` or no enabled language has matching filter changes
- **THEN** the workflow SHALL exit success without committing

### Requirement: Built-in paths-filter rules

The workflow SHALL ship a `dorny/paths-filter` configuration that maps each known language
to its dependency files; consumers SHALL NOT need to provide these. Initial mapping:

| Language | Filter paths |
| --- | --- |
| `go` | `go.mod`, `go.sum` |
| `python` (future) | `pyproject.toml`, `poetry.lock`, `requirements*.txt` |
| `rust` (future) | `Cargo.toml`, `Cargo.lock` |

#### Scenario: Filter does not match

- **WHEN** the PR changes only files outside any enabled language's filter paths
- **THEN** the workflow SHALL skip all generation steps and exit success

### Requirement: Single commit per run

The workflow SHALL produce **at most one** commit per invocation regardless of how many
languages or packages are processed, using the literal message `chore: update vendor hashes`
via `stefanzweifel/git-auto-commit-action@v7`.

#### Scenario: Changes across multiple languages

- **WHEN** both Go and Python (hypothetical) hashes change in the same run
- **THEN** the workflow SHALL create one commit containing both diffs with message
  `chore: update vendor hashes`

### Requirement: Fork PRs are skipped

The generate workflow SHALL be skipped on pull requests from forked repositories (no
`GHA_PAT_TOKEN` is available to push).

#### Scenario: Fork PR

- **WHEN** invoked from a fork PR
- **THEN** the workflow SHALL not run any generation steps; the final CI gate SHALL treat
  the `skipped` result as success

### Requirement: Language extension contract

Adding a new language (Python, Rust, Node, …) SHALL be additive: a new entry in the
language map inside `generate.yml` providing (1) a paths-filter rule, (2) a regenerate
command (e.g., `poetry lock`), and (3) a per-target "build / capture hash / rewrite
file" recipe. The per-target shape SHALL remain `{ "attr", "file" }` so the input
contract is uniform across languages; the language's recipe owns how `attr` is
translated into a build invocation (e.g., Go appends `.goModules`, Python may append
`.dependencies`, etc.).

#### Scenario: Adding Python support later

- **WHEN** a future contributor adds a Python entry to the language map
- **THEN** existing Go-only consumers SHALL continue to work unchanged; consumers
  opting in by adding `"python": {"targets":[...]}` to their `languages` input SHALL
  get Python hash regeneration without further input-shape changes
