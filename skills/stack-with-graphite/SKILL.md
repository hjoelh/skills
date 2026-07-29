---
name: stack-with-graphite
description: Analyze the current branch or a supplied GitHub pull request, split it into the smallest review-friendly stack, materialize the stack with the Graphite CLI, validate every branch, and submit the stacked pull requests. Use when the user asks to create, build, split, or submit a PR stack with Graphite or `gt`, rather than only planning one.
---

# Stack with Graphite

Turn existing work into a review-friendly Graphite stack and submit it as draft pull requests. Perform the work; do not stop after proposing a plan.

## Establish the source and prerequisites

1. Inspect repository instructions before changing anything.
2. Confirm `gt` is available and inspect `gt --version` plus the help for each mutating command before using it. Prefer current commands over remembered aliases.
3. Capture the recovery state with:
   - `git status --short`
   - `git branch --show-current`
   - `git rev-parse HEAD`
   - `git worktree list`
   - `gt trunk`
   - `gt log short --stack`
4. If the user supplies a PR number or URL, inspect it with `gh pr view` and `gh pr diff`, then use `gt get <pr-number>` when the stack is not already local.
5. Detect the repository's default or integration trunk. If Graphite is not initialized, run `gt init --trunk <branch>` only when that branch is unambiguous; otherwise ask the user.
6. Preserve unrelated work. Do not include, discard, stash, or rewrite changes outside the requested source.

Stop and report the exact prerequisite when `gt` is missing, Graphite authentication is required, the source is ambiguous, another worktree owns a branch that must be rewritten, or unrelated changes overlap the intended split.

## Design the smallest useful stack

Inspect the diff from trunk, commit history, tests, migrations, generated files, and dependency direction. Choose one PR when the work is already a coherent review unit.

Split only when every branch has an independently understandable purpose and the split materially reduces reviewer effort. Good boundaries include:

- mechanical setup or refactoring that enables later behavior;
- foundations required by later branches;
- separate user-visible behaviors;
- risky changes that deserve focused review.

Keep implementation with its required tests, migrations, generated output, and configuration when separating them would leave a misleading or broken intermediate state. Prefer a linear cumulative stack; Graphite stacks are dependent branches, not a collection of independent sibling PRs.

Before mutating, state the proposed ordered branches, contents, dependency direction, and validation. Use titles in the exact form `[STACK/<feature>] (n/total) Description`, with a short lowercase hyphenated feature name. Use `(1/1)` when no split is justified.

## Materialize with Graphite

Choose the least destructive workflow supported by the source:

- For an existing branch whose commit boundaries already match the plan, track it with `gt track --parent <parent>` when needed, then use `gt split --by-commit`.
- For file-aligned boundaries, use one or more `gt split --by-file <pathspec>` operations.
- For mixed changes in a single commit, use `gt split --by-hunk` interactively and assign every hunk deliberately.
- For uncommitted work based directly on the intended parent, stage only the first coherent slice with `gt add`, create it with `gt create <branch-name> -m "<commit-message>"`, then repeat from the new branch for each later slice.

Use Graphite commands for branch creation and dependency metadata. Do not create empty branches before adding their changes. Give each branch a concise lowercase hyphenated name and normally one atomic commit.

When splitting a branch that already has a PR, retain the original branch name for the slice that should preserve that PR and its review history. Never overwrite remote work that changed since it was last fetched. Do not use `--force`, `--no-verify`, delete branches, close PRs, or merge the stack unless the user explicitly requests it.

If a Graphite operation hits conflicts, inspect the conflict and resolve it only when the intended resolution is clear. Mark resolved files with `gt add` and continue with `gt continue`. Use `gt abort` when the correct resolution is uncertain; do not guess.

## Validate the stack

1. Run `gt restack`.
2. Inspect `gt log short --stack` and `gt log long`.
3. For every branch, inspect its parent-relative diff with `gt info --diff` or the current equivalent.
4. Confirm every file and hunk appears exactly once, each branch has the intended parent, and no unrelated work entered the stack.
5. Run the repository-required checks. Run focused tests on the branch that introduces the behavior and broader checks at the stack tip.
6. Re-check `git status --short`. Do not submit with unresolved changes or conflicts.

If validation reveals a bad boundary, fix the local stack with the appropriate Graphite operation and validate again before submission.

## Preview and submit

1. Run `gt submit --stack --dry-run` and review the exact branches and PR operations.
2. Submit new PRs as drafts with `gt submit --stack --draft --cli --edit`, preserving the planned titles and writing concise bodies that explain why each boundary exists and what changed.
3. Do not enable merge-when-ready or publish drafts unless the user explicitly asks.
4. Re-run `gt log short --stack` and collect the created or updated PR URLs from Graphite or `gh pr view`.

## Report

Return:

- the source branch or PR and trunk;
- the final ordered stack with branch names, exact PR titles, and parent relationships;
- the PR URLs and whether each is draft;
- validation run for each branch and any checks not run;
- any preserved uncommitted or unrelated work;
- any follow-up required from the user.

Do not claim a PR was created or updated unless submission succeeded and its URL was verified.
