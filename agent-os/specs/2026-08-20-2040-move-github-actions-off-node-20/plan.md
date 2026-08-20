# Move GitHub Actions off Node 20 action versions

FlowFast card 6839934 (Move GitHub Actions off Node 20 action versions).

## Context

GitHub Actions runners now force actions built for Node 20 onto Node 24 and emit a
deprecation warning. The warning is advisory while the shim exists; when GitHub removes it,
every affected step fails at once across the org, including image builds and deploys.

The warning was seen on the `build / build` job, which comes from the shared reusable
workflow `component-build.yml` in the `workflows` repo — so one fix there covers the whole
Eventide component fleet without touching those repos.

Verified 2026-08-20 by reading `action.yml` at each tag. **Two obvious-looking targets are
wrong:** `docker/build-push-action@v6` and `docker/setup-buildx-action@v3` are both still
`node20`. The correct targets are v7 and v4.

| Action | Pinned | Target |
|---|---|---|
| `docker/build-push-action` | v5 | **v7** |
| `docker/setup-buildx-action` | v3 | **v4** |
| `actions/checkout` | v3, v4 | **v6** |
| `actions/cache` | v4 | **v5** |
| `actions/setup-node` | v4 | **v6** |
| `actions/upload-artifact` | v4 | **v6** |
| `actions/setup-python` | v5 | **v7** |
| `actions/create-github-app-token` | v2 | **v3** |
| `slackapi/slack-github-action` | v2 | **v4** |
| `codecov/codecov-action` | v3, v4 | **v5** (composite) |

Already `node24`, no change: `ruby/setup-ruby@v1`, both `google-github-actions/*@v3`,
`anthropics/claude-code-action@v1`, `rubygems/release-gem@v1`.

Upstream-blocked: `codecov/test-results-action@v1` is `node20` with no `node24` release
(latest v1.2.1, checked 2026-08-20). Used by `hyerhub`. Carve-out agreed.

Constraint confirmed: build-push-action v7 and setup-buildx-action v4 both require Actions
Runner **≥ 2.327.1**. The ARC image tracks `ghcr.io/actions/actions-runner:latest` and
`infrastructure/github-actions/runner-version-cronjob.yaml` records GitHub refusing v2.334.0
around 2026-08-10, so the deployed runner clears the floor.

## Approach

Three phases by blast radius, plus a permanent fix so the drift does not return.

### Phase 0 — Spec folder and inventory

1. Save the spec folder into the `workflows` worktree at
   `agent-os/specs/2026-08-20-2040-move-github-actions-off-node-20/` (`plan.md`, `shape.md`,
   `standards.md`, `references.md`). Committed first, before any workflow edit.
2. Generate the authoritative affected-repo inventory: for each non-archived repo in the
   `hubbado` org, list the action pins under `.github/workflows`. Write it to a cached file
   so later steps do not re-hit the API — the earlier ad-hoc scan exhausted the 5,000/hr
   limit partway through.

### Phase 1 — `workflows` repo (highest leverage)

Worktree already created: `infrastructure/workflows/.worktrees/move-github-actions-off-node-20`,
branched off `origin/main` (that repo has no `master`).

Files, all under `.github/workflows/`:

- `component-build.yml` — `checkout@v4`→v6 (L49), `setup-buildx-action@v3`→v4 (L71),
  `build-push-action@v5`→v7 (L74), `cache@v4`→v5 (L95)
- `component-ci.yml` — `checkout@v4`→v6 (L57), `build-push-action@v5`→v7 (L76)
- `component-ci_no-docker.yml`, `component-package.yml`, `component-deploy.yml` — `checkout@v4`→v6,
  `cache@v4`→v5

Also add `.github/dependabot.yml` (template below), and `.gitignore` carrying `.worktrees/`.

**Before merging**, capture the cache baseline from the most recent pre-bump component build
run: total build step duration and the cached-layer count from the log. There is nothing to
compare against afterwards otherwise. `component-build.yml` pushes with
`cache-from`/`cache-to type=registry,ref=…:buildcache,mode=max`; `component-ci.yml` builds with
`load: true, push: false` and a bare-image-ref `cache-from`, and notably does **not** run
setup-buildx-action, so it relies on the runner's default buildx driver — the step most likely
to behave differently across two majors.

Merge, then re-run a component build and read the annotations.

### Phase 2 — deploy-bearing repos, by pull request

`hubbado_core`, `hyerhub`, `eco_core`, `eco_core-incident-southend`,
`one-drive-cv-library-es-ingest`, `kubernetes`, and `contributor-assets` itself. One branch per
repo carrying both the version bumps and `.github/dependabot.yml`; PR; merge on green.

`hyerhub` carries the most: `checkout@v4` ×6, `cache@v4` ×5, `upload-artifact@v4` ×3,
`setup-node@v4` ×2, the docker pair, `codecov-action@v4`, `create-github-app-token@v2`, plus the
blocked `test-results-action@v1`.

### Phase 3 — libraries, components and dormant repos, direct to master

Everything else: the `hubbado-*` gems, `libraries/*`, the component repos, `blog`,
`eco_landers`, `codecov-mcp`, `turnstile*`, `command-line-gem-generator`, `hubbado-sequence`,
`assignment_permission`, the `dev_*_service` repos. Most need only a single `checkout` bump.

New script in `contributor-assets`, following `update-ruby-version.sh`:

- iterate the Phase 0 inventory, not the hardcoded `projects/*.sh` arrays — those miss ~15
  affected repos (that gap is card 6839937)
- route every mutating command through `utilities/run-cmd.sh` so `DRY_RUN=true` genuinely
  dry-runs; `update-ruby-version.sh` bypasses it and pushes for real, which must not be copied
- `sed` the pins, write `.github/dependabot.yml`, commit, push to master
- reuse `utilities/boolean-env-var.sh` for the flags and `run-workflow.sh` to trigger CI fleet-wide

Dry-run first, read the diff, then run for real.

### Dependabot template

`.github/dependabot.yml`, identical in every repo:

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: monthly
    groups:
      github-actions:
        patterns:
          - "*"
```

## Verification

1. `bash contributor-assets/run-workflow.sh ci.yml` triggers CI across the Eventide fleet;
   `gh run view --log` on a completed run shows no "Node.js 20 is deprecated" annotation.
2. Re-run the Phase 0 inventory: no repo reports an action whose `action.yml` declares
   `node16` or `node20`, except `codecov/test-results-action@v1`.
3. `hubbado_core` and `hyerhub`: CI, image build and deploy green, images at the same registry
   paths as before.
4. Warm component build duration and cache-hit count no worse than the Phase 1 baseline.
5. Codecov coverage still reported on a PR after `codecov-action` v4→v5 — v5 changed token
   handling, so this may need a repo secret rather than being a pure version swap.
6. `gh api repos/{repo}/dependabot/alerts`-style check is not the signal; confirm instead that
   each repo shows the config on its Insights → Dependency graph → Dependabot tab, or that a
   grouped PR appears at the next monthly run.

## Out of scope

- `codecov/test-results-action@v1` — upstream-blocked, tracked on the card.
- Modernising the `contributor-assets` lists and `DRY_RUN` handling — card 6839937.
- The `--force-with-lease` push in `update-ruby-version.sh` — raised, deliberately kept.
