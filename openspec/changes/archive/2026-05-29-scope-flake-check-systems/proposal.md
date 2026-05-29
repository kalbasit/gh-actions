# Proposal: scope-flake-check-systems

## Why

The reusable `build.yml` (and the orchestrator's standalone `flake-check` job)
run the full `nix flake check` + coverage on **every** system in the matrix.
For consumers like ncps, the integration suite (Garage/Postgres/MySQL/Redis) is
architecture-independent, yet it runs a second time on the slower `aarch64-linux`
ARM runner where it both doubles wall-clock and flakes (Postgres timeouts). The
arch-specific artifact — the OCI image — is the only thing that genuinely needs
to be built per architecture.

## What Changes

- Add an opt-in `test_systems` input (JSON-array string, default `"[]"`) to
  `build.yml` and the orchestrator `ci.yml`.
- `"[]"` (default) preserves today's behavior: flake-check + coverage on every
  system. A non-empty list restricts those steps to the listed systems.
- The OCI image build/push (and the `oci-manifest` job) remain **per-system**
  regardless, so the multi-arch manifest is unchanged.
- The orchestrator forwards `test_systems` to `build.yml` and applies the same
  gate to its standalone `flake-check` job.

## Capabilities

### New Capabilities
- None.

### Modified Capabilities
- `oci-build`: `build.yml` scopes flake-check + coverage to `test_systems`
  while still building the image for every system.
- `reusable-ci`: the orchestrator gains the `test_systems` input, forwards it to
  `build.yml`, and applies it to the standalone `flake-check` job.

## Impact

- **Affected systems:** `.github/workflows/build.yml`, `.github/workflows/ci.yml`,
  and the `oci-build` + `reusable-ci` specs. No change to `generate.yml` or
  `openspec-guard.yml`.
- **Consumers:** backward compatible — default `"[]"` means existing consumers
  (signal-api-receiver, etc.) are unaffected until they opt in.
- **I/O / network / memory:** CI-only; opting in removes redundant ARM
  runner-minutes and one integration DB spin-up per excluded system. No runtime
  impact on any consumer's product.

## Non-goals

- Changing default behavior for existing consumers.
- Removing any architecture from the build matrix (images still built per arch).
- Reworking how a consumer's flake structures its checks/cohorts.
