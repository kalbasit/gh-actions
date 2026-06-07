## Context

`build.yml` is a `workflow_call` reusable workflow consumed by `ncps`, `swm`, and
`signal-api-receiver`. Its first step, **"Validate registry inputs"**, runs unconditionally
and currently asserts that `dockerhub_image` and `dockerhub_username` are either both set or
both empty:

```bash
dh_image_set=false; [[ -n "$DH_IMAGE" ]] && dh_image_set=true
dh_user_set=false;  [[ -n "$DH_USER"  ]] && dh_user_set=true

if [[ "$dh_image_set" != "$dh_user_set" ]]; then
  echo "::error::dockerhub_image and dockerhub_username must be set together ..."
  exit 1
fi

if [[ "$OCI_PUSH" == "true" ]]; then
  if [[ "$dh_image_set" != "true" && "$GHCR_ENABLED" != "true" ]]; then
    echo "::error::push_oci is true but no OCI registry is enabled ..."
    exit 1
  fi
fi
```

### Why it breaks on fork PRs

Consumers pass `dockerhub_image` as a **literal** (e.g. `kalbasit/ncps`) and
`dockerhub_username` from a **repository variable** (`${{ vars.DOCKERHUB_USERNAME }}`).
On a fork PR, GitHub withholds that variable, so the orchestrator forwards
`dockerhub_image="kalbasit/ncps"` but `dockerhub_username=""` → `image=true user=false`.
The caller (`ci.yml`) already forces `push_oci: false` for fork PRs, so the workflow is not
going to push anything — yet the unconditional pairing check still `exit 1`s, failing the
whole build leg in ~8s. This is the symptom in `ncps#1317`.

## Goals / Non-Goals

- **Goal:** Fork PRs (and any `push_oci: false` run) pass the validation step; non-fork
  pushing runs keep the exact same enforcement.
- **Non-Goal:** Changing inputs/secrets, the `ghcr_enabled` default, or any
  login/push/manifest step (all already gated on `push_oci`).

## Decision

Move the `dockerhub_image`/`dockerhub_username` pairing check **inside** the existing
`if [[ "$OCI_PUSH" == "true" ]]` block, alongside the "at least one registry" check:

```bash
if [[ "$OCI_PUSH" == "true" ]]; then
  if [[ "$dh_image_set" != "$dh_user_set" ]]; then
    echo "::error::dockerhub_image and dockerhub_username must be set together (got image=$dh_image_set user=$dh_user_set)"
    exit 1
  fi
  if [[ "$dh_image_set" != "true" && "$GHCR_ENABLED" != "true" ]]; then
    echo "::error::push_oci is true but no OCI registry is enabled (set dockerhub_image+dockerhub_username and/or ghcr_enabled)"
    exit 1
  fi
fi
```

### Why this is correct and safe

- When `push_oci == false`: validation is a no-op. Every step that consumes
  `dockerhub_username`/`dockerhub_token` (`Login to Docker Hub`, `Push the OCI image`, the
  `oci-manifest` job) is already `if: inputs.push_oci ...`, so a half-set pair can never reach
  an auth/push step.
- When `push_oci == true`: behavior is byte-for-byte identical to today — a misconfigured
  consumer (image without username, or vice-versa) still fails fast with the same message.

### Alternatives considered

- **Gate the whole step with a job-level `if`** — rejected: the "at least one registry"
  check must still run on real pushes; keeping one step with an internal guard is clearer.
- **Make the caller pass an empty `dockerhub_image` on forks** — rejected: pushes more
  conditional logic into every consumer; the fix belongs in the shared workflow.
- **Drop the pairing check entirely** — rejected: it catches genuine misconfiguration on
  pushing runs.

## Risks / Trade-offs

- Low risk; the change only relaxes a check on the non-pushing path. A consumer that sets
  only one of the pair will no longer get a warning on PR builds, but will still fail on the
  first push (branch/tag) build — acceptable, since pushes are where it matters.

## Verification

- `actionlint` clean on `build.yml`.
- Re-run CI on `ncps#1317` (a fork-style PR) once `build.yml@main` is updated: the
  `shared / build` legs go green; non-fork push builds on `main` remain unaffected.
