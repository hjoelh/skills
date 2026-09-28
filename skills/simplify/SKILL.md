---
name: simplify
description: Review changed code for reuse, quality, and efficiency with three parallel reviewers, then fix worthwhile findings. Use when the user asks for /simplify or wants to simplify and clean up recent changes.
---

# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix issues without changing intended behavior or expanding the task.

## Phase 1: Identify Changes

Run `git status --short` and `git diff` to see what changed. If there are staged changes, use `git diff HEAD` to include both staged and unstaged changes. Read any untracked files in scope too, since Git diffs omit them.

If there are no Git changes, review the most recently modified files that the user mentioned or that you edited earlier in the conversation. If no such files are identifiable, ask which files to review.

## Phase 2: Launch Three Review Agents in Parallel

Launch three review agents concurrently using the available subagent tool. Give each agent the full diff and the contents of any in-scope untracked files. For the no-diff fallback, give each agent the selected files. Keep agents read-only so fixes can be coordinated after all reviews finish.

If subagents are unavailable, perform the same three reviews yourself and disclose that limitation.

### Agent 1: Code Reuse Review

For each change:

1. Search for existing utilities and helpers that could replace newly written code. Use `rg` to find similar patterns in utility directories, shared modules, and adjacent files.
2. Flag new functions that duplicate existing functionality. Suggest the existing function to use instead.
3. Flag inline logic that could use an existing utility, such as hand-rolled string manipulation, manual path handling, custom environment checks, or ad-hoc type guards.

### Agent 2: Code Quality Review

Review the same changes for:

1. **Redundant state:** state that duplicates existing state, cached values that could be derived, or observers/effects that could be direct calls.
2. **Parameter sprawl:** adding parameters instead of generalizing or restructuring an existing function.
3. **Copy-paste with slight variation:** near-duplicate blocks that should share an abstraction.
4. **Leaky abstractions:** exposing internal details or breaking existing abstraction boundaries.
5. **Stringly-typed code:** raw strings where constants, enums, string unions, or branded types already exist.

### Agent 3: Efficiency Review

Review the same changes for:

1. **Unnecessary work:** redundant computations, repeated file reads, duplicate network/API calls, or N+1 patterns.
2. **Missed concurrency:** independent operations run sequentially when they could run in parallel.
3. **Hot-path bloat:** new blocking work in startup, per-request, or per-render paths.
4. **Unnecessary existence checks:** pre-checking file/resource existence before operating, creating a time-of-check/time-of-use race. Prefer operating directly and handling the error.
5. **Memory:** unbounded data structures, missing cleanup, or event listener leaks.
6. **Overly broad operations:** reading entire files when only a portion is needed, or loading all items when filtering for one.

## Phase 3: Fix Issues

Wait for all three reviews to finish. Aggregate and deduplicate their findings, then fix each worthwhile issue directly. Verify findings against the code before editing. If a finding is a false positive or not worth addressing, note it briefly and move on.

Preserve intended behavior and unrelated user changes. Run checks appropriate to the fixes and inspect the final diff.

Briefly summarize what was fixed, or confirm the code was already clean. Include validation results and any unresolved issues.
