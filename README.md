# auto-merge-workflow

A single, self-contained GitHub Actions workflow that **auto-merges PRs once
they're ready**, choosing the merge method from the base branch:

| Base branch | Merge method |
|---|---|
| `dev`  | squash |
| `main` | merge commit |

This toy repo exists to test the workflow before it ships to a real project.

## How it works

Add the **`ready-to-merge`** label to a PR targeting `dev` or `main`. The
workflow then merges it automatically once **all** of these gates pass:

1. **Every check run on the PR head is green** (or `skipped`/`neutral`).
2. **≥1 approving review**, and no `CHANGES_REQUESTED`.
3. **No unresolved review threads.**
4. **Mergeable.** If the head is `BEHIND` the base, the branch is auto-updated
   and re-evaluated. If it has conflicts (`DIRTY`), a comment is posted asking
   for manual resolution.

The `ready-to-merge` label is removed only on a **successful** merge, so its
presence on an open PR means "still waiting".

## Triggers

The workflow re-evaluates on any relevant change — you rarely need to do
anything beyond labelling:

- `ready-to-merge` label added
- new commit pushed (`synchronize`)
- draft → ready (`ready_for_review`)
- a review is submitted (approval also kicks off the merge)
- any check run completes (`check_run`)
- **manual**: comment `/merge` on the PR (admin/maintainer only), or run the
  workflow manually from the Actions tab with a PR number

## Error handling

- **Conflicts** → comment posted (once); fix, push, re-label or `/merge`.
- **Branch behind base** → auto-updated; if that fails, a comment is posted.
- **Merge command fails** (e.g. branch protection) → comment posted with the
  error; fix the cause and `/merge` to retry.
- **Checks fail** → nothing merges; push a fix and the next `check_run` retries.
- **Checks pending** → the workflow exits and is re-triggered when checks finish.

## Setup (one-time)

1. **Create the label**: `ready-to-merge`.
2. **Drop the workflow in**: copy `.github/workflows/auto-merge.yml` to your
   repo. The workflow uses the default `GITHUB_TOKEN` — no secrets needed.
3. **(Recommended) rulesets as a backstop** so the manual merge button can't
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

1. Create a feature branch off `dev`, push a change, open a PR to `dev`.
2. Add the `ready-to-merge` label.
3. Approve it (or have a maintainer approve).
4. Watch the workflow merge it (squash into `dev`).
5. Open a PR from `dev` to `main`, label + approve, and it merges as a merge
   commit.
