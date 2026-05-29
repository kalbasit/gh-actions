# Tasks: scope-flake-check-systems

- [x] 1. `build.yml`: add `test_systems` input (JSON array, default `"[]"`).
- [x] 2. `build.yml`: gate **Flake check**, **Build coverage**, and **Upload
  coverage to Codecov** with
  `inputs.test_systems == '[]' || contains(fromJson(inputs.test_systems), matrix.system)`
  (ANDed with the existing non-fork condition on the coverage steps).
- [x] 3. `build.yml`: leave the OCI image build/push steps and `oci-manifest`
  ungated (per-system).
- [x] 4. `ci.yml`: add `test_systems` input; forward to the `build` job; apply
  the gate to the standalone `flake-check` job.
- [x] 5. Spec delta `oci-build`: MODIFY `Reusable OCI build workflow` for the
  scoping + add the subset scenario.
- [x] 6. Spec delta `reusable-ci`: MODIFY orchestrator requirement + ADD
  `test_systems input` requirement.
- [x] 7. `openspec validate --change scope-flake-check-systems` passes.
- [x] 8. actionlint / `_lint-workflows.yml` clean on both workflows.
- [ ] 9. Commit on the feature branch and open a PR.
