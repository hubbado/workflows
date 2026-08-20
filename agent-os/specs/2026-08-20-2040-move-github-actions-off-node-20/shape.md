# Move GitHub Actions off Node 20 action versions — Shaping Notes

FlowFast card 6839934.

## Scope

Bump every GitHub Action pinned at a Node-16/20-era major across every non-archived repo in
the `hubbado` org, and add a dependabot `github-actions` config to each so the drift does not
return. Dormant repos included.

Three phases by blast radius: the shared reusable workflows first, then deploy-bearing repos
by pull request, then libraries and components through a fleet script.

## Decisions

- **`docker/build-push-action` targets v7, not v6, and `docker/setup-buildx-action` targets v4,
  not v3.** Both intermediate versions are still `node20`; the card originally named them and
  would have shipped a change that fixed nothing. Verified by reading `action.yml` at each tag.
- **Dormant repos are in scope** rather than excluded or archived. Uniform, no judgement calls
  per repo.
- **Dependabot ships in this card**, monthly interval, grouped so each repo raises one pull
  request rather than one per action.
- **Push style is split by blast radius.** Deploy-bearing repos get pull requests; libraries,
  components and dormant repos get commits pushed straight to master, matching the existing
  `update-ruby-version.sh` pattern.
- **`codecov/test-results-action@v1` is upstream-blocked** — `node20`, no `node24` release,
  latest v1.2.1 as of 2026-08-20. Carved out of the acceptance criteria rather than left as a
  criterion that can never pass.
- **The fleet script derives its repo list from the org**, not from the hardcoded arrays in
  `contributor-assets/projects/*.sh`, which miss roughly fifteen affected repos. Fixing those
  arrays is card 6839937 and is not done here.

## Context

- **Visuals:** None.
- **References:** See `references.md`.
- **Product alignment:** N/A — CI tooling, no product surface.

## Constraints surfaced during shaping

- `docker/build-push-action@v7` and `docker/setup-buildx-action@v4` both require Actions Runner
  **≥ 2.327.1**. The self-hosted ARC image tracks `ghcr.io/actions/actions-runner:latest`, and
  `infrastructure/github-actions/runner-version-cronjob.yaml` records GitHub refusing v2.334.0
  around 2026-08-10, so the deployed runner clears the floor.
- The `workflows` repo has no `master`; its default branch is `main`, which is also how callers
  reference it (`hubbado/workflows/.github/workflows/component-build.yml@main`).
- The org-wide inventory must be cached to a file. An ad-hoc scan during shaping exhausted the
  GitHub API rate limit of 5,000/hr partway through.
- `component-ci.yml` builds with `load: true, push: false` and does **not** run
  setup-buildx-action, so it depends on the runner's default buildx driver. It is the step most
  exposed to behaviour changes across two majors of build-push-action.

## Standards Applied

None. The `workflows` repo carries no `agent-os/standards/` directory, so there was no standards
index to draw from.
