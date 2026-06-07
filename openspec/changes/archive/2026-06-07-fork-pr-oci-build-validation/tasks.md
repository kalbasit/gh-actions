## 1. Fix build.yml registry-input validation

- [x] 1.1 In `.github/workflows/build.yml`, move the `dockerhub_image`/`dockerhub_username`
  pairing check (`dh_image_set != dh_user_set`) inside the existing
  `if [[ "$OCI_PUSH" == "true" ]]` block in the "Validate registry inputs" step, keeping the
  same error message; leave the "at least one registry enabled" check in place.
- [x] 1.2 Confirm no other step regresses: every `Login to Docker Hub`, `Push the OCI image`,
  and `oci-manifest` step/job remains gated on `inputs.push_oci` (no changes needed, verify).

## 2. Verify

- [x] 2.1 Run `actionlint .github/workflows/build.yml` (or `nix flake check` if it covers
  workflow lint) and confirm it is clean.
- [x] 2.2 Open the PR to `gh-actions` `main`; confirm the repo's own CI passes.

## 3. Deliver and archive

- [x] 3.1 Archive this OpenSpec change in the same PR (so `openspec-guard` passes) — sync the
  `oci-build` delta into `openspec/specs/oci-build/spec.md` and move the change to
  `openspec/changes/archive/`.
- [ ] 3.2 After merge, re-run CI on `kalbasit/ncps#1317` (which references `build.yml@main`)
  and confirm the `shared / build (x86_64-linux)` and `(aarch64-linux)` legs go green.
