---
name: code-standards-pr
description: Review the current GitHub pull request diff against the repository root AGENTS.md instructions using parallel file-group reviewers. Use when Codex needs to audit a PR or local branch changes for code-standard violations, repo-specific workflow mismatches, missing structural conventions, or implementation choices that conflict with the root repo guidance before review, merge, or handoff.
---

# Code Standards PR

Review the active PR like a code reviewer who is enforcing the repo's root written standards. Use parallel subagents to inspect grouped changed files, then return one collated review focused on concrete mismatches between the diff and the root `AGENTS.md`.

## Workflow

1. Resolve review context from the repo root.
2. Read the repository root `AGENTS.md` and ignore nested `AGENTS.md` files.
3. Inventory the changed files from the active PR or local branch diff.
4. Partition changed files into review groups.
5. Spawn parallel subagents, one per review group.
6. Collate the subagent results into one findings-first review.

## Resolve Review Context

Prefer GitHub PR context when available:

```bash
gh pr view --json number,title,baseRefName,headRefName,url
gh pr diff --name-only
gh pr diff
```

If `gh pr view` does not return active PR context, fall back to a local branch review:

1. Determine the best available local base branch.
2. Compute a local diff against that base.
3. Use the same grouped-subagent workflow and final output format as the PR path.

Use the final review to state whether the basis was a GitHub PR or local branch changes.

## Review Rules

- Treat this as a review task, not an implementation task.
- Prefer the actual PR base from `gh pr view` over assumptions like `origin/dev`.
- Use only the repository root `AGENTS.md`, even if nested `AGENTS.md` files exist elsewhere in the tree.
- Focus on violations that are visible in the diff or strongly implied by it.
- Quote or paraphrase the relevant rule closely enough that the reviewer can see why it applies.
- Call out missing tests only when the changed behavior or risk makes the gap meaningful.
- Avoid speculative findings that depend on unseen files or runtime behavior.
- Do not dump raw subagent notes into the final answer.

## File Grouping

Group changed files before dispatching reviewers.

- Keep groups small enough that one subagent can inspect the relevant hunks and immediate surrounding context without losing focus.
- Prefer meaningful boundaries over arbitrary chunking.
- Keep related implementation and test files together when practical.
- Keep obviously unrelated files in separate groups so findings stay precise.

If the PR is large, automatically reduce fanout by batching more aggressively. Do not stop to ask the user. The goal is broad standards coverage without creating excessive reviewer count or duplicate noise.

## Subagent Contract

Spawn one subagent per file group. Each subagent should:

- Review only its assigned file group plus any nearby code needed to confirm a standards violation.
- Use the repository root `AGENTS.md` as the sole standards source.
- Report only issues visible in the diff or strongly supported by immediately inspected context.
- Cite the affected file path for every finding.
- Quote or paraphrase the relevant `AGENTS.md` rule for every finding.
- Say `No findings` explicitly when nothing actionable is present.

## What To Check

- Component and hook structure required by the repo.
- Forbidden React patterns or APIs called out in `AGENTS.md`.
- Testing conventions, helper usage, and anti-patterns.
- File placement and import patterns.
- Workflow mismatches shown directly in the PR, such as generated code or test commands that violate repo guidance.

## Useful Commands

Use `gh pr diff` for the whole patch when PR context is available:

```bash
gh pr diff
```

Use `git diff` when you need a tighter view for one file after you know the real base:

```bash
git diff "$(git merge-base HEAD origin/<baseRefName>)"...HEAD -- path/to/file.tsx
```

For local fallback review, inspect changed files and patch from the local base:

```bash
git diff --name-only <base>...HEAD
git diff <base>...HEAD
```

If the PR base is not present as a local remote-tracking branch, fetch it or rely on `gh pr diff` for the patch view.

## Collation

After all subagents return:

- Merge duplicate or overlapping findings.
- Keep the clearest phrasing and highest severity when findings overlap.
- Drop raw `No findings` responses from the final summary.
- Order the final findings by severity.
- If nothing remains after collation, say explicitly that no standards violations were found.

## Output

- If there are findings, list them first with severity, file path, and a short explanation tied to the rule.
- If there are no findings, say so explicitly.
- Mention the review basis you used: PR number and base branch when available, otherwise the local diff base.
