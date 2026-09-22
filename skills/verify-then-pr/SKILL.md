---
name: verify-then-pr
description: Capture before and after screenshots, verify the changed behavior, then open a draft GitHub pull request with screenshots attached using GitHub CLI. Use when asked to verify work before creating a PR or to deliver a new draft PR with visual evidence.
---

# Verify then PR

Capture an honest before/after comparison, check the work, and deliver a draft PR with GitHub-hosted screenshots. Invoking this skill authorizes the scoped commit, push, draft creation, and evidence attachment. It does not authorize merging.

## 1. Establish the comparison

Read repository instructions and inspect the user's request, working-tree changes, and branch diff. Determine concrete acceptance criteria and the surface that demonstrates them: the application for integrated behavior, or Storybook for isolated component states.

Resolve the intended base branch, defaulting to the repository's default branch. Record the before revision and current working-tree state. Check for an existing PR for this head before creating anything; reuse an existing draft instead of creating a duplicate. If it is already ready for review, report that conflict rather than silently changing its state.

Check `gh auth status` and `gh pr edit --help` for `--attach`. The native syntax is `gh pr edit ... --attach`, not a standalone `github cli attach` or `gh attach` command. If unavailable, report the CLI upgrade or authentication requirement; do not silently substitute Cloudflare, release assets, or committed images.

## 2. Capture before

When implementation has not started, capture the relevant state before editing. When changes already exist, run the pre-change revision in an isolated temporary worktree, usually the merge base with the intended PR base. Do not reset, overwrite, or stash away the user's work to obtain a screenshot.

Use the same route, data, interaction state, theme, and viewport for the before/after pair. For responsive UI, capture desktop at 1440 × 900 and mobile at 390 × 844 CSS pixels unless the task specifies different sizes. Record the revision, route, viewport, and any fixtures or mocks alongside the local evidence.

Keep screenshots outside tracked source files. Inspect them for readability and sensitive data before upload. Use representative nonsensitive data. If a baseline cannot run or the relevant state cannot be reproduced, report the missing evidence and its cause; never label a screenshot of the changed code as “before.” Complete independent checks, but do not claim this workflow is complete without the required pair.

For genuinely nonvisual work, use a meaningful visible result when one exists. If screenshots cannot demonstrate the change, explain that limitation and ask whether check output may replace visual evidence rather than manufacturing UI proof.

## 3. Verify and capture after

Complete any implementation already authorized by the user. Review the final diff for unintended changes, run the repository's required checks and focused checks for the acceptance criteria, then exercise the changed behavior on the chosen surface. Screenshots alone do not prove interactions or persistence.

Fix failures within scope and rerun affected checks. If a required check remains failing or blocked, report the exact failure and stop before PR creation unless the user explicitly authorizes an incomplete draft.

Capture the verified after state using the same setup as before. Open and inspect every final image. Recapture unreadable images; add a detail crop if the changed area is too small. Add concise arrows or numbered callouts only when they help explain a subtle difference, without covering relevant content. Preserve the unannotated originals, and never alter the UI result to imply behavior that did not occur.

Remove temporary capture overrides before the final checks and commit. Disclose fixtures or mocks and their coverage limits. If the implementation changes after capture or checks, repeat the affected verification and screenshots so the evidence matches what will be pushed.

## 4. Open the draft PR

If the `pr` skill is installed, read and follow it for Git and draft creation after verification succeeds. This skill adds the required evidence section to its concise summary. Do not invoke `verify-pr`: that separate workflow requires an existing PR and uses Cloudflare uploads.

If `pr` is unavailable, use this self-contained fallback:

1. Reuse a suitable feature branch, or create `codex/<topic>` from the current state when detached or on an integration branch.
2. Stage and commit only changes belonging to this task using plain Git. Keep screenshots and unrelated work out of the commit.
3. Push the feature branch with upstream, without force-pushing.
4. Write the PR body to a temporary Markdown file. Lead with why the change is needed and summarize the resulting behavior. Include actual verification results, limitations, the verified commit SHA, and labeled before/after images. Do not add a test plan.
5. Run `gh pr create --draft --base <base> --head <head> --title <title> --body-file <body-file>`.

After creation, register the returned PR URL with the task's artifact attachment tool when available. Keep the PR a draft.

## 5. Attach screenshots and check delivery

Use GitHub CLI's native `--attach` flag to attach the images to the new draft PR. Read the latest PR body first and preserve its summary when adding evidence. In the body file, use the exact local paths passed to `--attach`; GitHub CLI replaces those image references with uploaded URLs.

Example evidence section, replacing paths with the actual evidence files:

```markdown
## Verification

Verified commit: FULL_COMMIT_SHA

Checks: actual commands and exercised behavior, with any limitations.

### Before

![Before: describe the original behavior](/absolute/evidence/before.png)

### After

![After: describe the verified change](/absolute/evidence/after.png)
```

```bash
gh pr edit <pr-url> --body-file <complete-body-file> \
  --attach /absolute/evidence/before.png \
  --attach /absolute/evidence/after.png

gh pr view <pr-url> --json url,isDraft,baseRefName,headRefName,headRefOid,body
```

Repeat `--attach` for additional viewport pairs. If creating the PR directly, the same flags can be included on `gh pr create --draft`.

Confirm the PR is a draft, the base/head are correct, and `headRefOid` matches the verified commit. Inspect the posted body for uploaded URLs in place of local paths, and open the PR to confirm each image renders legibly with the correct before/after label.

An attachment failure can still create or update the PR with a subset of the images. After any nonzero exit, inspect the existing PR and body before retrying. Preserve successful uploads and retry only missing images; never create a second PR as an upload retry. If still blocked, report the existing draft URL and exactly which evidence is missing.

Return the draft PR URL, a brief verification result, any material limitation, and the local evidence directory. Keep local evidence available for the user to inspect. Do not claim completion until both the draft and the attached evidence are verified.

Reference: [GitHub CLI attachment documentation](https://docs.github.com/en/github-cli/github-cli/attaching-files-with-github-cli).
