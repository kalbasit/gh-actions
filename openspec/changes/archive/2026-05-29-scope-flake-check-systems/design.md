# Design: scope-flake-check-systems

## Context

`build.yml` runs a matrix over `inputs.systems` and, on each leg, runs
`nix flake check -L` → build coverage → codecov upload → build/push OCI image.
The orchestrator `ci.yml` also has a standalone `flake-check` matrix job for the
`oci: false` path. The flake check and coverage are architecture-independent for
typical Go consumers, so running them on every arch is redundant cost. This repo
is shared, so any change must default to current behavior.

## Goals / Non-Goals

**Goals**
- Let a caller restrict flake-check + coverage to a subset of `systems`.
- Keep the OCI image per-arch so multi-arch manifests stay complete.
- Zero behavior change for callers that don't set the new input.

**Non-Goals**
- Auto-detecting the canonical arch; opt-in only.
- Touching tag generation, manifest assembly, or fork-safety logic.

## Decisions

### D1 — `test_systems` JSON-array input, default `"[]"` = all
A JSON-array string mirrors the existing `systems` input. Empty array is the
"run everywhere" sentinel (backward compatible). The gate on each test/coverage
step is:

```
inputs.test_systems == '[]' || contains(fromJson(inputs.test_systems), matrix.system)
```

`contains(array, item)` is a native GitHub Actions expression function, so no
shell step is needed. For coverage/codecov, this ANDs with the existing
non-fork condition.

*Alternatives:* a boolean `skip_aarch64_tests` (arch-specific, inexpressive);
defaulting to the first system (surprising for multi-system consumers). Rejected.

### D2 — OCI image build stays ungated
`nix build .#packages.<system>.docker` builds the consumer's image package,
which does not depend on `nix flake check`. Leaving it ungated means every arch
still produces its image; the `oci-manifest` job continues to combine
`<tag>-<system>` images for all `systems`. Only the test/coverage steps are
gated.

### D3 — Plumb through the orchestrator
`ci.yml` adds `test_systems` and forwards it to the `build` job. It also applies
the gate to its own standalone `flake-check` job (the `oci: false` path) so the
behavior is consistent regardless of which path a consumer uses.

## Risks / Trade-offs

- **A consumer sets `test_systems` to a system not in `systems`.** → The
  `contains` gate simply never matches that leg; no error, the step is skipped.
  Documented in the input description.
- **Coverage now comes from a subset of arches.** → Intended; coverage is
  arch-independent. Consumers that genuinely need per-arch coverage keep the
  default `"[]"`.

## Migration Plan

1. Land this change on `main`; `@main` consumers pick it up.
2. Existing consumers see no change (default `"[]"`).
3. ncps opts in via `test_systems: '["x86_64-linux"]'` in its caller.
4. **Rollback:** consumers drop the input; the field is inert when unused.

## Open Questions

- None blocking. A future change could let the orchestrator default
  `test_systems` to the native arch, but that is out of scope here.
