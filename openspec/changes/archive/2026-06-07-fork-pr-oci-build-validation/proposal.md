## Why

Fork PRs (and any PR where the `dockerhub_username` repo variable is not exposed) fail
the reusable `build.yml` workflow at its **"Validate registry inputs"** step, even though
the build is correctly configured to skip pushing on forks. This blocks CI on consumer PRs
(e.g. `kalbasit/ncps#1317`) with the spurious error
`dockerhub_image and dockerhub_username must be set together (got image=true user=false)`,
directly contradicting the existing `oci-build` "PR from a fork" scenario, which says a fork
PR SHALL still build the image (read-only) and SHALL skip registry login/push.

## What Changes

- In `build.yml`, the **paired-registry validation** (`dockerhub_image` and
  `dockerhub_username` must both be set or both unset) is gated behind `push_oci == true`.
  When `push_oci` is false — fork PRs (the caller forces it false), or any non-pushing run —
  an asymmetric pair is tolerated because none of the login/push/manifest steps run.
- The "at least one registry enabled" check is unchanged (already gated on `push_oci`).
- No change to workflow inputs, secrets, or the owner/push code paths: when `push_oci` is
  true, the pairing requirement is still enforced exactly as before.
- The `oci-build` spec's "PR from a fork" scenario is tightened to state explicitly that
  registry-input validation MUST NOT fail the workflow when pushing is disabled.

## Capabilities

### New Capabilities

- _None._

### Modified Capabilities

- `oci-build`: clarify that the build job's registry-input validation only enforces the
  `dockerhub_image`/`dockerhub_username` pairing when `push_oci` is true, so fork PRs (where
  the username variable is stripped) still build cleanly.

## Impact

- **Repo (this):** `.github/workflows/build.yml` — one conditional moved/added in the
  "Validate registry inputs" step.
- **Consumers:** `ncps`, `swm`, `signal-api-receiver` — fork-PR CI is unblocked once they
  reference the updated `build.yml@main`. No consumer-side change required (their
  `dockerhub_image` literal + `dockerhub_username` variable pairing is already correct for
  non-fork runs).
- **Inputs/secrets:** unchanged. **Non-fork push behavior:** unchanged.
- **Delivery:** PR directly to `gh-actions` `main` (CI-infra fix); the OpenSpec change is
  archived in the same PR to satisfy `openspec-guard`.
