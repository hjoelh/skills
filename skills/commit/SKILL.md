---
name: commit
description: Instantly stage and commit all Git changes in the current repository with maximum speed. Use when the user asks for /commit, wants a fast full-repo commit, or wants everything staged and committed without being asked for a commit message.
---

# Commit

Stage every tracked and untracked change in the current Git repository, then commit immediately with an empty message if needed.

## Workflow

1. Confirm the current working directory is inside a Git repository with `git rev-parse --show-toplevel`.
2. Stage everything with `git add -A`.
3. Commit immediately with `git commit --allow-empty --allow-empty-message -m ""`.
4. Share the commit result and the new commit hash.

## Guardrails

- Stage all changes in the repo with `git add -A`; do not try to select files unless the user asks.
- Optimize for speed over narration; do not ask for a commit message.
- Do not amend, rebase, force-push, or rewrite history unless the user explicitly asks.
- Allow empty commits when there are no staged changes.
- If Git blocks the commit because user name or email is missing, report that exact problem.

## Commands

```bash
git rev-parse --show-toplevel
git add -A
git commit --allow-empty --allow-empty-message -m ""
git rev-parse --short HEAD
```
