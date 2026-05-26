## 1. Fix filter paths in ci.yml

- [x] 1.1 In `.github/workflows/ci.yml`, change the builtin `go_deps` filter from `["go.mod", "go.sum"]` to `["**/go.mod", "**/go.sum"]`

## 2. Fix filter paths in generate.yml

- [x] 2.1 In `.github/workflows/generate.yml`, change the builtin `go_deps` filter from `["go.mod", "go.sum"]` to `["**/go.mod", "**/go.sum"]`

## 3. Sync spec

- [x] 3.1 Run `openspec sync --change vendor-hash-not-working` to apply the delta spec to `openspec/specs/dependency-hash-generation/spec.md`
