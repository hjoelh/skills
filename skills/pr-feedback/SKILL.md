---
name: pr-feedback
description: "Use when triaging review comments on a PR from any source — CodeRabbit, Graphite Reviewer, human engineers. Infers current PR from branch or accepts a PR number as argument."
allowed-tools: Bash, Read, Edit, Write, AskUserQuestion, Glob, Grep, Task
---

# PR Review Triage

Fetch ALL review comments on a PR and walk through each one interactively. Handles bot and human comments, deduplicates overlapping bot feedback, and provides deep analysis for architectural discussions.

## 1. Identify the PR

If an argument was provided, use it as the PR number. Otherwise detect from the current branch:

```bash
gh pr view --json number --jq '.number'
```

## 2. Fetch all comments

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

Fetch from three sources:

### 2a. Inline review comments (all reviewers)

```bash
gh api "repos/$REPO/pulls/$PR/comments" --paginate
```

This returns comments from ALL reviewers — CodeRabbit, Graphite, humans.

### 2b. Review bodies (for CodeRabbit outside-diff-range comments)

```bash
gh api "repos/$REPO/pulls/$PR/reviews" --paginate
```

CodeRabbit embeds "Outside diff range" comments in review bodies because GitHub doesn't allow inline comments outside the diff. Extract these from the **last** review body containing them (earlier ones are superseded).

### 2c. PR issue comments

```bash
gh api "repos/$REPO/issues/$PR/comments" --paginate
```

Issue comments often contain CI/deploy noise and non-actionable summaries, but some review agents post actionable feedback there. Fetch them and filter carefully so Claude, Cursor, human, or other review-agent feedback is triaged alongside inline review comments and review bodies.

## 3. Filter and build threads

### Filter out

- **CI/deploy bots**: `vercel[bot]`, `github-actions[bot]`, `linear[bot]`
- **CodeRabbit summaries and walkthroughs**: Review bodies that don't contain "Outside diff range", CodeRabbit issue-comment walkthroughs, and other non-actionable summary-only comments
- **Non-actionable issue comments**: Status updates, generated summaries, release/deploy links, test output, or comments without a concrete question, concern, or requested change
- **Pure affirmations**: Comments with no actionable content — "Neat", "Great stuff", "Nice one, thank you!", approval-only reviews with empty body

### Include from issue comments

- **Review-agent feedback**: Actionable issue comments from `claude[bot]`, `cursor[bot]`, CodeRabbit, Graphite, or other review agents when they identify a concrete concern, ask a review question, or suggest a code/design change
- **Human review feedback**: Human issue comments that ask review questions, request changes, raise architectural concerns, or include specific code suggestions
- **Mixed summary + action comments**: Include only the actionable portions when a comment combines a summary with concrete feedback

### Build reply threads

Group comments by `in_reply_to_id` to reconstruct conversation threads. A thread is: the original comment + all replies. The entire thread is essential context — it shows whether the issue has already been discussed, agreed upon, or dismissed.

**Skip comments already resolved.** If a thread shows the concern has been handled, skip it. Look for:

- CodeRabbit's "✅ Addressed in commit" markers
- Author or any team member replying to dismiss the concern (e.g., "purposely not erroring on read path", "It's ok bro, it exists already")
- Mutual agreement to defer (e.g., "let's leave for now since it's a reversible decision")
- GitHub's resolved conversation state (if using review threads)

## 4. Categorize and deduplicate

Dispatch a subagent (Task tool, subagent_type: "general-purpose", model: "haiku") with all remaining comments. Send each comment as a JSON object with: `id`, `source_type` (`inline_review`, `review_body`, or `issue_comment`), `author`, `path` (nullable), `line` (or line range, nullable), `body`, `is_bot` (boolean), `thread` (array of replies with author and body). The subagent does two things:

### 4a. Categorize each comment

| Category                | Source               | Characteristics                                     |
| ----------------------- | -------------------- | --------------------------------------------------- |
| `bot-suggestion`        | Review agents        | Has concrete code fix (suggestion/diff block)       |
| `bot-observation`       | Review agents        | Flags issue but no concrete fix                     |
| `human-code-suggestion` | Human                | Includes a specific code change                     |
| `human-architectural`   | Human                | High-level — about approach, design, tradeoffs      |
| `human-question`        | Human                | Asks a question (ends with `?` or is interrogative) |
| `human-nitpick`         | Human                | Naming, style, or minor convention feedback         |

### 4b. Deduplicate overlapping bot comments

When review agents flag the same issue on the same or overlapping lines in the same file, or in issue comments that clearly reference the same concern, merge into one item:

- Keep the more complete suggestion
- Note which bots flagged it
- Be strict — only merge when they describe the same concrete issue, not just the same general category

### 4c. Group bot comments needing the same fix

Bot comments that require the same fix applied in multiple places (e.g., "add error handling to `mapRow`" in 5 call sites) should be grouped together. Be strict — only group when the fix is essentially identical.

Return JSON:

```json
[
  {
    "group_label": "Silent JSON unmarshal error in mapper.go",
    "category": "bot-suggestion",
    "sources": ["coderabbit", "graphite"],
    "items": [
      {
        "id": 123,
        "source_type": "inline_review",
        "path": "mapper.go",
        "line": 62,
        "severity": "major",
        "title": "JSON unmarshal error silently ignored",
        "description": "...",
        "suggestion": "...",
        "thread": [
          {
            "author": "paulc1204",
            "body": "purposely not erroring on read path"
          }
        ]
      }
    ]
  }
]
```

## 5. Parse suggestions by source

### CodeRabbit

- **Severity**: First line — `_🟡 Minor_`, `_🟠 Major_`, or `_🔴 Critical_`
- **Title**: Bold text after first blank line
- **Inline suggestions**: Code in ` ```suggestion ` blocks — exact replacement for referenced lines
- **Outside-diff suggestions**: Code in ` ```diff ` blocks inside `✏️ Suggested fix` details — use `-` lines to find old text, `+` lines for replacement
- **AI Agent Prompt**: Text in `🤖 Prompt for AI Agents` details section (use as guidance)

### Graphite Reviewer

- **Description**: Plain text problem statement
- **Suggestion**: Code in ` ```suggestion ` blocks (same format as GitHub suggestions)
- **Attribution**: Ends with "_Spotted by Graphite Agent_"

### Claude, Cursor, and other issue-comment review agents

- **Actionability**: Include only concrete findings, review questions, requested changes, or code/design suggestions
- **Location**: Use referenced file paths, line numbers, symbols, or quoted snippets when present; leave `path` and `line` null if the comment is PR-level but still actionable
- **Suggestion**: Preserve code blocks, bullet recommendations, or explicit "should" / "consider" changes as the proposed solution context

### Human comments

No structured format. Check for code blocks (specific suggestion), questions, or references to other files/patterns.

## 6. Walk through each item

Present items in this order:

1. **Human comments first** — these need human-to-human attention
2. **Bot suggestions** grouped by file
3. **Bot observations**

If a comment has an existing thread, always show the full thread so the user sees what's already been discussed.

---

### For `bot-suggestion` (and `human-code-suggestion`)

Present: source(s), file, line, severity, explanation, suggested code change.

Options (AskUserQuestion):

- **Review solution** — Show the proposed change for all locations in the group and ask before applying
- **Review alternate fix** — Propose another way to address the issue and ask before applying
- **Ignore** — Dismiss (will reply with reason)

### For `bot-observation`

Present: what was identified, relevant code context.

Options:

- **Review fix** — Propose a fix and ask before applying
- **Defer** — Acknowledge as future work
- **Ignore** — Dismiss

### For `human-architectural`

**This requires deeper analysis before presenting options.**

1. Read the relevant code and its surrounding context
2. Understand what the reviewer is suggesting as an alternative
3. Identify the tradeoffs — why was the current approach chosen? What would change with the reviewer's suggestion?
4. If there's an existing thread, factor in what's already been discussed

Present:

- Reviewer's feedback (quote it)
- The relevant code
- **Tradeoff analysis** — concisely explain both sides: current approach pros/cons vs reviewer's suggestion pros/cons
- Any existing thread discussion

Options:

- **Review implementation** — Propose the change and ask before applying
- **Agree but defer** — Acknowledge, note for future (draft reply for user approval)
- **Discuss** — Draft a reply exploring the tradeoff (user reviews before posting)
- **Push back** — Draft a reply explaining why current approach is preferred (user reviews before posting)
- **Already addressed** — Point to where it was handled

### For `human-question`

1. Read the relevant code to understand context
2. Formulate a proposed answer

Present: the question, relevant code, your proposed answer.

Options:

- **Reply with answer** — Post the proposed answer (user can edit first)
- **Let me handle this** — User will reply manually

### For `human-nitpick`

Present: the feedback and current code.

Options:

- **Review change** — Show the proposed change and ask before applying
- **Ignore** — No action

## 7. Act on decisions

### Applying code changes

- Before using Edit or Write, present the exact intended change when possible. If the exact patch is not practical to show, present a concise solution summary with the affected files and lines. Wait for the user to approve that proposed solution before mutating files.
- Read the file and apply using the Edit tool
- **`suggestion` blocks**: Exact replacement text for the referenced lines
- **`diff` blocks**: Match `-` lines to find old text, replace with `+` lines
- Apply all locations in a group before moving to the next item

### Replying to comments

**Inline comments** (have a comment ID):

```bash
gh api "repos/$REPO/pulls/$PR/comments/$COMMENT_ID/replies" -f body="$REPLY"
```

**Outside-diff-range comments** (no individual comment ID — reply to the review):

```bash
gh api "repos/$REPO/pulls/$PR/reviews/$REVIEW_ID/comments" \
  -f body="Regarding $PATH lines $LINES: $REPLY" \
  -f path="$PATH" -f line=1 -f side="RIGHT"
```

**Issue comments** (PR-level comments):

```bash
gh api "repos/$REPO/issues/$PR/comments" -f body="$REPLY"
```

### Ignore reasons (bot comments)

When ignoring bot comments, ask why (AskUserQuestion):

- "Intentional design choice"
- "Not applicable here"
- "Will address in a follow-up"
- (Other — free text)

Reply with the reason. For human comments, always reply — don't leave them without a response.

### Drafting replies

Keep replies concise and direct. Match the team's style — informal, no fluff. Always show the draft to the user for approval before posting.

## 8. Summary

After all items are processed:

- Comments reviewed: N (CodeRabbit: X, Graphite: Y, Human: Z)
- Skipped (already resolved / affirmations): N
- Deduplicated: N bot comments merged into M items
- Accepted / Fixed differently / Replied / Deferred / Ignored
- Files modified
- Replies posted

If changes were made, ask the user if they want to commit.
