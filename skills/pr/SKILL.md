---
name: pr
description: Create a draft GitHub pull request from the current git state into the repository's default branch using the gh CLI. Use when the user asks to open or create a PR, wants the base branch to match the repo's main/default branch automatically, wants a short PR summary, or explicitly says to omit a test plan.
---

# PR

Create the draft pull request with `gh`, targeting the repository's default branch unless the user explicitly asks for a different base. Reuse a suitable feature branch when one already exists, otherwise create a PR branch first.

## Workflow

1. Inspect the current git state with `git branch --show-current`, `git rev-parse --abbrev-ref HEAD`, and `git status --short`.
2. If there are unstaged, staged, or uncommitted repo changes that belong in the PR, stage and commit them before opening the PR.
3. Resolve the PR head branch:
   - If `git branch --show-current` returns a non-default feature branch, use it.
   - If the repo is on detached HEAD, create and switch to a feature branch from the current commit before continuing.
   - If the current branch is a base or integration branch such as `main`, `master`, or `dev`, create and switch to a feature branch before continuing.
4. Determine the default base branch in this order:
   - `git symbolic-ref --quiet --short refs/remotes/origin/HEAD` and strip the `origin/` prefix.
   - `git remote show origin` and read the `HEAD branch` line.
   - `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.
5. Summarize the branch diff briefly so the PR title and body match the actual changes.
6. Write a concise PR body with a short summary only. Do not add a `Test plan` section unless the user explicitly asks for one.
7. Create the draft PR with `gh pr create --draft --base <default-branch> --head <current-branch> --title <title> --body <body>`.
8. Share the draft PR URL and the exact base/head branches used.

## Guardrails

- Prefer the repo's detected default branch over assuming `main`.
- If the repo is on detached HEAD, create a feature branch from the current commit instead of stopping.
- If the current branch is the default branch or another integration branch like `dev`, create a feature branch before opening the PR.
- If the current branch is already a suitable feature branch, reuse it instead of creating a new one.
- If the PR-worthy changes are not committed yet, create a commit before opening the PR.
- If `gh pr create` fails because the branch is not pushed, push the current branch with upstream and retry with `--draft`.
- Keep the title and body tight; default to 1 short title and 2-4 body bullets or sentences.
- Always open the PR as a draft, even if the user does not explicitly ask for one.
- Do not include a test plan, checklist, or boilerplate footer unless the user asks.
- If there are no committed or staged changes relative to the base branch, warn the user before creating an empty PR.

## Commands

```bash
git branch --show-current
git rev-parse --abbrev-ref HEAD
git status --short
git checkout -b codex/<topic>
git switch -c codex/<topic>
git add <files>
git commit -m "<message>"
git symbolic-ref --quiet --short refs/remotes/origin/HEAD
git remote show origin
gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
git log --oneline <base>..HEAD
git diff --stat <base>...HEAD
gh pr create --draft --base <base> --head <head> --title "<title>" --body "<body>"
```
