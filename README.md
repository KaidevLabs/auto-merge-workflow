# auto-merge-workflow

A single, self-contained GitHub Actions workflow that **auto-merges PRs once
they're ready**, choosing the merge method from the base branch:

| Base branch | Merge method |
|---|---|
| `dev`  | squash |
| `main` | merge commit |

This toy repo exists to test the workflow before it ships to a real project.

## How it works

Any **non-draft** PR targeting `dev` or `main` is merged automatically once
**all** of these gates pass:

1. **Every check run on the PR head is green** (or `skipped`/`neutral`).
2. **≥ `AUTO_MERGE_MIN_APPROVALS` approving reviews** (repo variable, default
   `1`), and no `CHANGES_REQUESTED`.
3. **No unresolved review threads.**
4. **Mergeable.** If the head is `BEHIND` the base, the branch is auto-updated
   and re-evaluated. If it has conflicts (`DIRTY`), a comment is posted asking
   for manual resolution.

There is **no opt-in label**: an approved, green, non-draft PR just merges.
To **hold** a PR, add the **`no-auto-merge`** label (opt-out).

## Triggers

The workflow re-evaluates on any relevant change, so you rarely need to do
anything:

- a review is submitted (approval kicks off the merge)
- any check run completes (`check_run`)
- new commit pushed (`synchronize`)
- draft → ready (`ready_for_review`), reopened, labelled/unlabelled
- **manual**: comment `/merge` on the PR (admin/maintainer only), or run the
  workflow manually from the Actions tab with a PR number

## Error handling

- **Conflicts** → comment posted (once); fix, push, or comment `/merge`.
- **Branch behind base** → auto-updated; if that fails, a comment is posted.
- **Merge command fails** (e.g. branch protection) → comment posted with the
  error; fix the cause and `/merge` to retry.
- **Checks fail** → nothing merges; push a fix and the next `check_run` retries.
- **Checks pending** → the workflow exits and is re-triggered when checks finish.

## Setup (one-time)

1. **Create the `no-auto-merge` label** (optional — the hold escape hatch).
2. **Drop the workflow in**: copy `.github/workflows/auto-merge.yml` to your
   repo. It uses the default `GITHUB_TOKEN` — no secrets needed.
3. **(Optional) set `AUTO_MERGE_MIN_APPROVALS`** repo variable to change the
   number of required approvals (default `1`). Set to `0` in a toy repo to test
   the mechanics without a second approver.
4. **(Recommended) rulesets as a backstop** so the manual merge button can't
   bypass the gates. In Settings → Rules → Rulesets, for `main` and `dev`:
   - `pull_request` rule: `required_approving_review_count: 1`,
     `required_review_thread_resolution: true`, and `allowed_merge_methods`:
     `["merge"]` for `main`, `["squash"]` for `dev`.
   - `required_status_checks`: your CI job names.

## Caveat

Pushes made by the default `GITHUB_TOKEN` do **not** trigger other workflow
runs. Merging works fine, but if you add CI that must run on `push` to
`dev`/`main` after an auto-merge, trigger that CI from the `pull_request`
event or use a PAT/GitHub App token for the merge step.

## Testing in this repo

The branch layout used here: `main` (default), `dev`, and feature branches.

```
feature/x  --(PR, squash)-->  dev  --(PR, merge commit)-->  main
```

`AUTO_MERGE_MIN_APPROVALS` is set to `0` here so PRs merge without an approval
(you can't approve your own PRs). Open a PR to `dev`, mark it ready, and the
workflow merges it as a squash; open a PR from `dev` to `main` and it merges as
a merge commit.
