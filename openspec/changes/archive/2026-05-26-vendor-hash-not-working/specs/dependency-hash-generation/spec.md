## MODIFIED Requirements

### Requirement: Built-in paths-filter rules

The workflow SHALL ship a `dorny/paths-filter` configuration that maps each known language
to its dependency files; consumers SHALL NOT need to provide these. Initial mapping:

| Language | Filter paths |
| --- | --- |
| `go` | `**/go.mod`, `**/go.sum` |
| `python` (future) | `pyproject.toml`, `poetry.lock`, `requirements*.txt` |
| `rust` (future) | `Cargo.toml`, `Cargo.lock` |

The `go` filter paths SHALL use the `**/` prefix so that `go.mod`/`go.sum` files at
any depth in the repository (including subdirectory Go modules) trigger `generate`.

#### Scenario: Filter does not match

- **WHEN** the PR changes only files outside any enabled language's filter paths
- **THEN** the workflow SHALL skip all generation steps and exit success

#### Scenario: Subdirectory Go module change triggers generate

- **WHEN** a PR changes `plugins/forge-github/go.mod` (a nested Go module) but NOT the
  root `go.mod`
- **THEN** the `go_deps` filter SHALL return `true` and the `generate` job SHALL run
