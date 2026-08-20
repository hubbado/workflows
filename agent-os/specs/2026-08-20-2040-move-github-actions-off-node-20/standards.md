# Standards for Move GitHub Actions off Node 20 action versions

No standards were applied. The `workflows` repo carries no `agent-os/standards/` directory, so
there is no standards index for this repo to draw from.

The conventions this work follows instead come from the repo itself and from
`contributor-assets`:

- Reusable workflows are referenced by callers as
  `hubbado/workflows/.github/workflows/<name>.yml@main`. The default branch is `main`, not
  `master`.
- Action versions are pinned to a major tag (`@v6`), not a commit SHA. This work keeps that
  convention rather than introducing SHA pinning.
- Fleet-wide changes go through `contributor-assets`, are dry-runnable via `DRY_RUN`, and route
  mutating commands through `utilities/run-cmd.sh`.
