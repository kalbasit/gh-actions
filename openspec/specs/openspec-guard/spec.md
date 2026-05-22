# openspec-guard Specification

## Purpose
TBD - created by archiving change shared-workflows. Update Purpose after archive.
## Requirements

### Requirement: OpenSpec active-change guard

The repository SHALL provide a reusable workflow at
`.github/workflows/openspec-guard.yml` that fails when any non-archive subdirectory exists
under the consumer's OpenSpec changes path (default `openspec/changes`), forcing
contributors to archive completed changes before merging.

#### Scenario: Active change present

- **WHEN** the consumer repo contains `openspec/changes/some-change/` (any non-archive
  directory at depth 1)
- **THEN** the workflow SHALL write a job summary listing each active change directory and
  SHALL fail with `::error::Active OpenSpec changes must be archived before merging — see
  job summary for details`

#### Scenario: No active changes

- **WHEN** `openspec/changes/` contains only the `archive/` directory (or is absent)
- **THEN** the workflow SHALL print "No active OpenSpec changes" and exit success

### Requirement: Configurable path

The workflow SHALL accept a `path` input (default `openspec/changes`) so consumers using a
non-default OpenSpec layout can point at their changes directory.

#### Scenario: Custom path

- **WHEN** invoked with `path: docs/openspec/changes`
- **THEN** the workflow SHALL scan `docs/openspec/changes` instead of the default
