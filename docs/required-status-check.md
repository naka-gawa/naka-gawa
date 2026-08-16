# CI: single aggregated required status check

This repository uses a single required status check to gate pull requests,
instead of registering every individual CI job in the branch ruleset.

Reference: [How to manage GitHub Actions required status checks](https://zenn.dev/shunsuke_suzuki/articles/how-to-manage-github-actions-required-status-check)
by Suzuki Shunsuke.

## Problem

GitHub branch rulesets require you to list each status check that must pass
before a pull request can be merged. Managing that list directly has two
recurring problems:

- **The list drifts.** Every time a CI job is added, renamed, or removed, the
  ruleset has to be updated by hand. If someone forgets, a failing job no
  longer blocks merges, or a required check that no longer exists blocks every
  merge.
- **Workflow-level path filters break required checks.** A workflow skipped by
  a top-level `paths:` filter never reports its status check. GitHub then waits
  for a check that will never arrive, permanently blocking the merge. In this
  repository the daily automated profile-update pull requests only change files
  under `profile-3d-contrib/` and never touch `.github/**`, so a path-filtered
  lint workflow would be skipped and its required check would never complete.

## Design

### One aggregated status check

`.github/workflows/ci.yaml` is the only workflow that runs on pull requests. It
contains a single `status-check` job that depends on every other CI job and
succeeds only when all of them succeed:

```yaml
status-check:
  runs-on: ubuntu-latest
  needs: [lint]
  if: always()
  steps:
    - name: Fail if any needed job did not succeed
      if: contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled')
      run: exit 1
```

- The job always runs (`if: always()`) so it reports an explicit **success
  (green)** check rather than relying on GitHub's "a skipped required check
  passes" behavior.
- When every needed job succeeds (or is skipped), the failing step is itself
  skipped and the job succeeds, so the pull request is mergeable.
- When any needed job fails or is cancelled, the step runs `exit 1` and the job
  fails, blocking the merge.

Only `status-check` is registered in the branch ruleset. Because it aggregates
results through `needs`, the ruleset never needs to change when the set of CI
jobs changes.

### Jobs live in a reusable workflow

The actual CI jobs live in the reusable workflow
`.github/workflows/workflow_call_lint.yaml`, which `ci.yaml` calls via
`uses:`. Because `status-check` depends on that single call, every job inside
the reusable workflow is automatically covered.

**Add new pull-request CI jobs to the reusable workflow, not to `ci.yaml`.** A
job added directly to `ci.yaml` would run without being aggregated into
`status-check`, so it would not block merges.

### Job-level path filtering

Instead of a workflow-level `paths:` filter, filtering is done at the job level
with `dorny/paths-filter`. The workflow therefore always runs and always
reports `status-check`, while the expensive lint job runs only when workflow
files change:

```yaml
changes:
  outputs:
    workflows: ${{ steps.filter.outputs.workflows }}
  steps:
    - uses: dorny/paths-filter@... # pinned
      id: filter
      with:
        filters: |
          workflows:
            - '.github/**/*.yml'
            - '.github/**/*.yaml'

pinact:
  needs: [changes]
  if: needs.changes.outputs.workflows == 'true'
  # ...
```

For a pull request that does not change any workflow files (such as the daily
profile update), the `pinact` job is skipped, the reusable workflow succeeds,
`status-check` is skipped, and the required check passes.

## Configuring the branch ruleset

In the branch ruleset for `main`, under **Require status checks to pass**,
register only:

- `status-check`

Do not register the individual jobs (`pinact`, `changes`, etc.). Adding or
renaming jobs in the reusable workflow requires no ruleset change.

## Auto-merge

Both the daily contribution-stats update PR and Renovate dependency PRs are
merged automatically. GitHub native auto-merge waits for the required
`status-check` to pass before merging, so it only works once the required
status check above is configured in the ruleset.

- **Contribution stats update PR** (`.github/workflows/generate-profile`): the
  workflow enables auto-merge on the PR it opens via
  `peter-evans/enable-pull-request-automerge`.
- **Renovate PRs** (`.github/renovate.json`): `automerge` with
  `platformAutomerge` lets Renovate enable GitHub native auto-merge on the PRs
  it opens. Major updates are excluded (`matchUpdateTypes: ["major"]`,
  `automerge: false`) so they are merged manually after review.

### Auto-approving automated PRs

`.github/CODEOWNERS` (`* @naka-gawa`) makes a code-owner review required on
every PR. A bot (Renovate, or the App that opens the daily profile-update PR)
cannot approve its own PRs, and — because a GitHub App or `github-actions[bot]`
cannot be listed as a code owner — neither can satisfy a code-owner review. So
the approving review has to be submitted **as the code owner** (a token owned by
`@naka-gawa`).

Rather than store that token in this repository, approval uses the
[`csm-actions/approve-pr-action`](https://github.com/csm-actions/approve-pr-action)
**Client/Server model**, so the token never lives in the client repo:

1. **Client** (`.github/workflows/approve-automated-pr.yaml`, this repo): on a
   bot-authored PR it runs the action in client mode, which asks the server
   repository to approve by creating an `approve-pr-*` label there. It uses a
   client GitHub App and never touches the approving token.
2. **Server** (`approve-pr-server`, a separate private repo): a workflow
   triggered by that label validates the PR — all commits signed and linked to
   users, committer in `allowed_committers` — then approves it with the code
   owner's PAT, which is stored **only** in the server repo (as a protected
   environment secret).

Because the PAT is centralized in the server repo, the same server can approve
PRs for any number of client repositories. **Which** automated PRs get approved
is defined in `.github/approve-pr.yaml` (not in the workflow) — the workflow
reads that list and passes it to the action. Add the committer of another
automated PR — e.g. the daily profile update — to that file to cover it too, no
workflow change needed. Major Renovate updates are still `automerge: false` in
`renovate.json`, so they never auto-merge and a human merges them manually.

#### One-time setup

A **single approver GitHub App** (one App per purpose) is shared by the client
and server workflows: `issues: write` (create the label on the server repo),
`pull requests: read` and `contents: read` (validate PRs). It has **no**
`pull requests: write`, so it cannot approve on its own. Install it on this repo
and the server repo, webhook disabled.

Client repo (this repo) — Actions variable/secret:

- `APPROVE_PR_APP_ID` (variable) and `APPROVE_PR_PRIVATE_KEY` (secret): the
  approver App above.

Server repo (`approve-pr-server`):

- The server workflow (`label: created` → `csm-actions/approve-pr-action` in
  server mode).
- `APPROVE_PR_APP_ID` / `APPROVE_PR_PRIVATE_KEY`: the same approver App.
- A protected environment secret holding **@naka-gawa's fine-grained PAT**
  scoped to the client repos with only `pull requests: write`. This is the only
  place the PAT is stored, and it is what actually submits the approval.

## Files

| File | Role |
| --- | --- |
| `.github/workflows/ci.yaml` | Pull-request entry point; defines the aggregated `status-check` job. |
| `.github/workflows/workflow_call_lint.yaml` | Reusable workflow holding the actual CI jobs. Add new jobs here. |
| `.github/renovate.json` | Renovate config; enables auto-merge for dependency PRs. |
| `.github/workflows/approve-automated-pr.yaml` | Client half of the code-owner auto-approval for automated (bot) PRs. |
| `.github/approve-pr.yaml` | List of committers whose automated PRs are auto-approved. |
