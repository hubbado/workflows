# References for Move GitHub Actions off Node 20 action versions

## Similar Implementations

### Fleet-wide file edit across every repo

- **Location:** `contributor-assets/update-ruby-version.sh`
- **Relevance:** The house pattern for applying one mechanical change to every repo at once —
  loop the project list, edit in place, `git add`, commit, push to master.
- **Key patterns:** Iterate a sourced array; `sed -i ''` for the in-place edit; check
  `git status --porcelain` before committing so untouched repos are skipped.
- **What not to copy:** It calls `git`, `sed` and `echo` directly rather than through
  `utilities/run-cmd.sh`, so `DRY_RUN=true` has no effect and the run pushes for real. It also
  pushes with `--force-with-lease`.

### Dry-run-aware command execution

- **Location:** `contributor-assets/utilities/run-cmd.sh` and
  `contributor-assets/utilities/boolean-env-var.sh`
- **Relevance:** Already implements the dry-run behaviour the new script needs; no reason to
  write another.
- **Key patterns:** `run-cmd "$cmd"` prints under `DRY_RUN=true` and `eval`s otherwise;
  `boolean-env-var NAME` normalises yes/no/true/false/1/0 flags.

### Copying one file into every repo

- **Location:** `contributor-assets/utilities/update-file.sh`
- **Relevance:** Closest existing analogue to dropping `.github/dependabot.yml` into each repo.
- **Key patterns:** `FORCE` distinguishes "update only where it exists" from "create anywhere";
  it checks out master, commits, pushes, then restores the previously checked-out branch.

### Triggering a workflow across the fleet to verify

- **Location:** `contributor-assets/run-workflow.sh`
- **Relevance:** The verification step — runs one named workflow on every Eventide project.
- **Key patterns:** `gh workflow list -R hubbado/$project --json path` to check the workflow
  exists before `gh workflow run -R hubbado/$project -r master`.

### Repos already on the target versions

- **Location:** `hubbado_saas/hubbado_core/.github/workflows/ci.yml`
- **Relevance:** Already on `checkout@v6`, `cache@v5`, `setup-node@v6`, `upload-artifact@v6`.
  Shows the target pins working against this codebase's CI, so those bumps carry little risk.
- **Note:** Its `docker-image.yml` is still on the docker pair, which is the untested part.

## Upstream sources read

- `action.yml` at each candidate tag, for the `runs.using` value. The only reliable way to tell
  which majors are `node24`; release notes and the marketplace do not state it.
- `docker/build-push-action` v6.0.0 and v7.0.0 release notes — v7 removes
  `DOCKER_BUILD_NO_SUMMARY` and `DOCKER_BUILD_EXPORT_RETENTION_DAYS`, and requires Actions
  Runner ≥ 2.327.1.
- `docker/setup-buildx-action` v4.0.0 release notes — removes deprecated inputs and outputs,
  same runner floor.
- GitHub changelog, deprecation of Node 20 on Actions runners:
  https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/

## Related cards

- **6839937 (Make contributor-assets fleet scripts cover the whole org)** — the stale project
  arrays and the `DRY_RUN` gap found while shaping this work.
