# auto-merge-workflow

A single, self-contained GitHub Actions workflow that **arms GitHub's native
auto-merge** on pull requests, choosing the merge method from the base branch:

| Base branch | Merge method |
|---|---|
| `dev`  | squash |
| `main` | merge commit |

The branch **ruleset is the single source of truth for gating** — required
checks, required approvals, and review-thread resolution are all enforced by
GitHub, not re-implemented in the workflow. This toy repo exists to test the
workflow before it ships to a real project.

## How it works

The workflow does **not** evaluate checks/approvals/threads itself. On a
relevant PR event it:

1. **Maps the base branch to a merge method** (`dev` → squash, `main` → merge
   commit); PRs to other branches are skipped.
2. **Skips what can't be merged**: drafts, cross-repo (fork) PRs, and PRs with
   conflicts (a comment explains each). A branch that is **behind** its base is
   updated automatically.
3. **Arms GitHub auto-merge** (`gh pr merge --auto`).

From there GitHub takes over: it waits until the base branch's ruleset is
satisfied (required checks green, enough approvals, threads resolved, branch up
to date) and merges — reactively, with no polling.

> **Why native auto-merge?** GitHub does **not** fire `check_run`/`check_suite`
> events for checks created by GitHub Actions (an anti-recursion guard), so a
> self-contained "wait for checks, then merge" workflow is never re-triggered
> when the tests finish. Arming auto-merge delegates the waiting and merging to
> GitHub, which already knows how to do both.

## Commands

Comment on a PR:

- `/merge` — re-evaluate and re-arm auto-merge (admin/maintainer).
- `/merge-force` — merge immediately, bypassing the ruleset (admin only; only
  works if the ruleset has **bypass actors** configured — see Setup).

You can also re-run the workflow manually from the Actions tab with a PR
number.

## Comments (kept minimal)

The bot only comments when a human must act, plus one orientation message:

- **Intro** (once per PR): how it works, plus the *live* ruleset config for the
  base branch (approvals required, required checks, merge method), and a loud
  warning if the repo's **Allow auto-merge** setting is off.
- **Action-required** (posted once each): merge conflicts, fork PR (needs an
  admin), a behind-base branch that couldn't be auto-updated, and arm /
  force-merge failures.

It stays silent for pending checks, waiting-for-approval, and "already armed" —
GitHub's own UI (the merge box, the Checks tab) already shows those.

## Setup (one-time)

1. **Enable auto-merge on the repo** (required): Settings → General → Pull
   Requests → **Allow auto-merge**. Without this, `gh pr merge --auto` cannot
   arm anything.
2. **Drop the workflow in**: copy `.github/workflows/auto-merge.yml` to your
   repo. It uses the default `GITHUB_TOKEN` — no secrets needed. (Set an
   `ADMIN_TOKEN` secret only if you want `/merge-force`.)
3. **Configure the rulesets** — these are the actual gates. In Settings → Rules
   → Rulesets, for `main` and `dev`:
   - **Require a pull request before merging**: set
     `required_approving_review_count` (e.g. `1`) and enable
     **Require review thread resolution**.
   - **Require status checks to pass**: add your CI check name(s) (e.g.
     `test-summary`), and optionally **Require branches to be up to date**.
   - **Allowed merge methods** must include what the workflow picks:
     `squash` (for `dev`) and `merge` (for `main`).
   - **(Optional) Bypass actors**: add admins here if you want `/merge-force`
     (or any admin override) to be able to bypass the ruleset. With no bypass
     actors, the ruleset applies to *everyone*, including admins.

## Caveat

Pushes made by the default `GITHUB_TOKEN` do **not** trigger other workflow
runs. Merging works fine, but if you add CI that must run on `push` to
`dev`/`main` after an auto-merge, trigger that CI from the `pull_request`
event or use a PAT/GitHub App token for the merge step.

## Testing in this repo

Branch layout: `main` (default), `dev`, and feature branches.

```
feature/x  --(PR, squash)-->  dev  --(PR, merge commit)-->  main
```

Both `dev` and `main` have a ruleset requiring 1 approval, resolved threads,
and the `test-summary` check. Open a PR to `dev` and the workflow arms a
**squash** auto-merge; approve it and GitHub merges once `test-summary` is
green. Open a PR from `dev` to `main` and it arms a **merge-commit**
auto-merge instead.
