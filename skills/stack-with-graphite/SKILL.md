---
name: stack-with-graphite
description: Analyze the current branch or a supplied GitHub pull request, propose the smallest review-friendly stack, wait for the user to confirm the plan, then materialize and submit it with the Graphite CLI. Use when the user wants to plan a PR stack and execute the approved plan with Graphite or `gt`.
---

# Stack with Graphite

Plan a review-friendly stack first. Do not mutate git state, Graphite metadata, or GitHub until the user explicitly confirms the proposed plan. After confirmation, implement that exact plan with the Graphite CLI and submit it as draft pull requests.

## Phase 1: Plan without making changes

### Identify the source

- Inspect repository instructions.
- If the user supplies a PR number or URL, inspect its title, base, head, files, commits, and diff with `gh pr view` and `gh pr diff`. Do not check it out or run `gt get` yet.
- Otherwise inspect the current branch, status, detected default or integration base, commit history, and diff from the base.
- Preserve uncommitted work in the analysis and distinguish it from committed changes.
- If the source is ambiguous, state the assumption and identify the exact branch or PR analyzed.

### Decide whether to split

First determine whether one PR is the clearest review unit. Recommend one PR when the work is tightly coupled, cannot be validated independently, or splitting would add coordination without reducing reviewer effort.

Split only when each proposed PR has a coherent purpose and the split materially improves reviewability. Good boundaries include:

- independently understandable setup, refactoring, or mechanical changes;
- a foundation required by later behavior;
- separate user-visible behaviors or components;
- risky changes that deserve focused review.

Keep coupled implementation, required tests, migrations, generated output, and configuration together when separating them would leave an incomplete or misleading intermediate state. Do not create extra PRs merely to increase the count.

Prefer cumulative dependent branches because Graphite models a stack through parent relationships. Recommend independent branches only when the slices are genuinely non-overlapping; explain that those branches are not one Graphite stack.

### Present the plan

Return:

1. A source summary identifying the branch or PR, base, scope, and whether uncommitted work is included.
2. The recommended number of PRs and why the work should or should not be split.
3. An ordered table or list where every PR includes:
   - a title in the exact form `[STACK/<feature>] (n/total) Description`;
   - a proposed lowercase hyphenated branch name;
   - its purpose and why the boundary improves reviewability;
   - included files or change groups;
   - its base, parent, dependencies, and merge order;
   - intermediate-state constraints;
   - validation and focused review notes.
4. Why this is the smallest review-friendly decomposition and why obvious alternative splits were rejected.

Derive `<feature>` from the work using a short lowercase hyphenated label. Use `(1/1)` when one PR is correct.

End by asking the user to confirm the proposed plan. Until they explicitly confirm, do not create or switch branches, commit, restack, push, modify the source, initialize Graphite, fetch a PR with Graphite, or open PRs.

## Phase 2: Execute only after confirmation

Treat an explicit approval such as “confirm,” “approved,” “go ahead,” or “submit it” in response to the plan as authorization to materialize and submit that plan. If the user requests changes, revise the plan and ask for confirmation again. Do not interpret the initial invocation of this skill as confirmation.

### Establish Graphite prerequisites

1. Confirm `gt` is available and inspect `gt --version` plus the help for each mutating command before using it. Prefer current commands over remembered aliases.
2. Capture the recovery state with `git status --short`, `git branch --show-current`, `git rev-parse HEAD`, `git worktree list`, `gt trunk`, and `gt log short --stack`.
3. If the confirmed source is a supplied PR that is not local, use `gt get <pr-number>`.
4. Detect the repository's default or integration trunk. If Graphite is not initialized, run `gt init --trunk <branch>` only when that branch is unambiguous; otherwise ask the user.
5. Preserve unrelated work. Do not include, discard, stash, or rewrite changes outside the confirmed plan.

Stop and report the exact prerequisite when `gt` is missing, Graphite authentication is required, another worktree owns a branch that must be rewritten, or unrelated changes overlap the intended split.

### Materialize with Graphite

Choose the least destructive workflow supported by the source:

- For an existing branch whose commit boundaries already match the plan, track it with `gt track --parent <parent>` when needed, then use `gt split --by-commit`.
- For file-aligned boundaries, use one or more `gt split --by-file <pathspec>` operations.
- For mixed changes in a single commit, use `gt split --by-hunk` interactively and assign every hunk deliberately.
- For uncommitted work based directly on the intended parent, stage only the first coherent slice with `gt add`, create it with `gt create <branch-name> -m "<commit-message>"`, then repeat from the new branch for each later slice.

Use Graphite commands for branch creation and dependency metadata. Do not create empty branches before adding their changes. Give each branch a concise lowercase hyphenated name and normally one atomic commit.

When splitting a branch that already has a PR, retain the original branch name for the slice that should preserve that PR and its review history. Never overwrite remote work that changed since it was last fetched. Do not use `--force`, `--no-verify`, delete branches, close PRs, or merge the stack unless the user explicitly requests it.

If a Graphite operation hits conflicts, inspect the conflict and resolve it only when the intended resolution is clear. Mark resolved files with `gt add` and continue with `gt continue`. Use `gt abort` when the correct resolution is uncertain; do not guess.

### Validate the stack

1. Run `gt restack`.
2. Inspect `gt log short --stack` and `gt log long`.
3. For every branch, inspect its parent-relative diff with `gt info --diff` or the current equivalent.
4. Confirm every file and hunk appears exactly once, each branch has the intended parent, and no unrelated work entered the stack.
5. Run the repository-required checks. Run focused tests on the branch that introduces the behavior and broader checks at the stack tip.
6. Re-check `git status --short`. Do not submit with unresolved changes or conflicts.

If validation reveals a bad boundary, fix the local stack with the appropriate Graphite operation and validate again before submission.

### Preview and submit

1. Run `gt submit --stack --dry-run` and review the exact branches and PR operations.
2. Submit new PRs as drafts with `gt submit --stack --draft --cli --edit`, preserving the planned titles and writing concise bodies that explain why each boundary exists and what changed.
3. Do not enable merge-when-ready or publish drafts unless the user explicitly asks.
4. Re-run `gt log short --stack` and collect the created or updated PR URLs from Graphite or `gh pr view`.

### Report

Return:

- the source branch or PR and trunk;
- the final ordered stack with branch names, exact PR titles, and parent relationships;
- the PR URLs and whether each is draft;
- validation run for each branch and any checks not run;
- any preserved uncommitted or unrelated work;
- any follow-up required from the user.

Do not claim a PR was created or updated unless submission succeeded and its URL was verified.
