---
name: pr
description: Create a GitHub pull request from the current branch into the repository's default branch using the gh CLI. Use when the user asks to open or create a PR, wants the base branch to match the repo's main/default branch automatically, wants a short PR summary, or explicitly says to omit a test plan.
---

# PR

Create the pull request with `gh`, targeting the repository's default branch unless the user explicitly asks for a different base.

## Workflow

1. Confirm the current branch with `git branch --show-current`.
2. Determine the default base branch in this order:
   - `git symbolic-ref --quiet --short refs/remotes/origin/HEAD` and strip the `origin/` prefix.
   - `git remote show origin` and read the `HEAD branch` line.
   - `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.
3. Summarize the branch diff briefly so the PR title and body match the actual changes.
4. Write a concise PR body with a short summary only. Do not add a `Test plan` section unless the user explicitly asks for one.
5. Create the PR with `gh pr create --base <default-branch> --head <current-branch> --title <title> --body <body>`.
6. Share the PR URL and the exact base/head branches used.

## Guardrails

- Prefer the repo's detected default branch over assuming `main`.
- If the current branch already matches the default branch, stop and report that a feature branch is needed before opening a PR.
- If `gh pr create` fails because the branch is not pushed, push the current branch with upstream and retry.
- Keep the title and body tight; default to 1 short title and 2-4 body bullets or sentences.
- Do not include a test plan, checklist, or boilerplate footer unless the user asks.
- If there are no committed or staged changes relative to the base branch, warn the user before creating an empty PR.

## Commands

```bash
git branch --show-current
git symbolic-ref --quiet --short refs/remotes/origin/HEAD
git remote show origin
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
gh pr create --base <base> --head <head> --title "<title>" --body "<body>"
```
