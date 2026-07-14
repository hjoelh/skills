---
name: stack
description: Analyze the current branch or a GitHub pull request and plan a review-friendly stack of new pull requests, splitting work only when the boundaries materially improve reviewability. Use when a change may benefit from multiple ordered PRs.
---

# Stack

Plan a stack of new pull requests from the current branch or a supplied GitHub PR. This skill is plan-only: do not create or switch branches, commit, push, modify the source branch, or open PRs.

## Identify the source

- If the user supplies a PR number or URL, use that PR's diff as the source. Inspect its title, base branch, head branch, changed files, commits, and diff with `gh pr view` and `gh pr diff`.
- Otherwise inspect the current branch with `git branch --show-current`, `git status --short`, the repository's detected default or integration base, the commit history, and the diff from the base.
- Preserve any uncommitted changes in the analysis and clearly distinguish them from committed changes.
- If the source is ambiguous, state the assumption and identify the exact branch or PR analyzed.

## Decide whether to split

First determine whether one PR is the clearest review unit. Recommend one PR when the changes are tightly coupled, cannot be validated independently, or splitting would add coordination without reducing reviewer effort.

Split only when each proposed PR has a coherent purpose and the split materially improves reviewability. Good boundaries can include:

- independently understandable setup, refactoring, or mechanical changes;
- a foundation that enables a later behavior change;
- separate user-visible behaviors or components;
- risky changes that deserve focused review;
- tests or migration work that can be reviewed independently without obscuring the production change.

Keep coupled implementation, required tests, migrations, and configuration together when separating them would make an intermediate PR incomplete, misleading, or unreviewable. Do not create extra PRs merely to increase the count.

## Choose the stack shape

Choose the structure that best matches the change and explain the choice:

- Use cumulative dependent branches when later work naturally builds on earlier work or sequencing gives reviewers a smaller incremental diff. Each item targets the previous item as its base, and the merge order is explicit.
- Use independent branches when slices are genuinely non-overlapping and can safely target the same base. State any ordering constraints even when the branches are independent.

Every item represents a new PR. Never recommend editing or replacing the supplied PR; if it is part of the source context, describe the proposed new PRs separately.

## Required output

Return a concrete stack plan with:

1. A short source summary identifying the branch or PR, base, scope, and whether uncommitted work is included.
2. The recommended number of PRs and why the work should or should not be split.
3. An ordered table or list where every PR includes:
   - a title in the exact form `[STACK/<feature>] (n/total) Description`;
   - a concise purpose and rationale for the boundary;
   - the included files or change groups;
   - its base/head relationship and dependencies;
   - merge order and any intermediate-state constraints;
   - validation and focused review notes.
4. A brief statement explaining why the proposed split is the smallest review-friendly decomposition, including why obvious alternative splits were rejected.

Derive `<feature>` from the work, using a short lowercase hyphenated label. Use `(1/1)` when the correct recommendation is a single PR.

## Guardrails

- Do not perform any git or GitHub write operation.
- Do not assume that a large diff must be split; prioritize coherent review units over PR count.
- Do not hide dependencies between stack items.
- Do not claim that files or commits belong to an item without inspecting the source diff or history.
- If the source cannot be inspected, report the missing context and provide only a conditional planning framework rather than inventing a stack.
