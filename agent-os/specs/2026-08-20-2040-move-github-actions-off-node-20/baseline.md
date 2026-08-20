# Pre-bump build baseline

Captured 2026-08-20, before the action version bumps merged. Compare a post-merge component
build against these numbers.

## Source run

- Repo: `hubbado/timesheet_component`
- Workflow: Build (calls `hubbado/workflows/.github/workflows/component-build.yml@main`)
- Run: 28683095459, 2026-07-03T20:58Z, conclusion success
- Action versions at the time: `checkout@v4`, `setup-buildx-action@v3`, `build-push-action@v5`,
  `cache@v4`

## Timings

| Step | Duration |
|---|---|
| `build / build` job, end to end | 68s |
| Set up Docker Buildx | 10s |
| Build image and push to Docker (to GCP) | 45s |
| Cache imagetag | 1s |

## Cache behaviour

- Cached layers reported by buildkit: **1**
- `importing cache manifest from …/timesheet-component:buildcache` — present
- `exporting cache to registry` — present
- Image pushed to `…/timesheet-component:latest` and the timestamped `master-…` tag

## Deprecation state

The log carries **1** "Node.js 20 is deprecated" annotation. A post-merge run of the same
workflow must carry none.

## Note on comparability

The fleet's most recent component builds are from 2026-07-03, so no build is warm in the sense
of "ran minutes ago". The registry buildcache is what makes a run warm here, and it persists,
so the comparison holds as long as the post-merge run builds the same component from an
unchanged Dockerfile.
