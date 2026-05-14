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

Fetch from two sources:

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

**Do NOT fetch** issue comments (`/issues/{PR}/comments`) — those are summaries, walkthroughs, and CI output, not actionable review feedback.

## 3. Filter and build threads

### Filter out

- **CI/deploy bots**: `vercel[bot]`, `github-actions[bot]`, `linear[bot]`
- **CodeRabbit summaries**: Review bodies that don't contain "Outside diff range"
- **Pure affirmations**: Comments with no actionable content — "Neat", "Great stuff", "Nice one, thank you!", approval-only reviews with empty body

### Build reply threads

Group comments by `in_reply_to_id` to reconstruct conversation threads. A thread is: the original comment + all replies. The entire thread is essential context — it shows whether the issue has already been discussed, agreed upon, or dismissed.

**Skip comments already resolved.** If a thread shows the concern has been handled, skip it. Look for:

- CodeRabbit's "✅ Addressed in commit" markers
- Author or any team member replying to dismiss the concern (e.g., "purposely not erroring on read path", "It's ok bro, it exists already")
- Mutual agreement to defer (e.g., "let's leave for now since it's a reversible decision")
- GitHub's resolved conversation state (if using review threads)

## 4. Categorize and deduplicate

Dispatch a subagent (Task tool, subagent_type: "general-purpose", model: "haiku") with all remaining comments. Send each comment as a JSON object with: `id`, `author`, `path`, `line` (or line range), `body`, `is_bot` (boolean), `thread` (array of replies with author and body). The subagent does two things:

### 4a. Categorize each comment

| Category                | Source               | Characteristics                                     |
| ----------------------- | -------------------- | --------------------------------------------------- |
| `bot-suggestion`        | CodeRabbit, Graphite | Has concrete code fix (suggestion/diff block)       |
| `bot-observation`       | CodeRabbit, Graphite | Flags issue but no concrete fix                     |
| `human-code-suggestion` | Human                | Includes a specific code change                     |
| `human-architectural`   | Human                | High-level — about approach, design, tradeoffs      |
| `human-question`        | Human                | Asks a question (ends with `?` or is interrogative) |
| `human-nitpick`         | Human                | Naming, style, or minor convention feedback         |

### 4b. Deduplicate overlapping bot comments

When CodeRabbit and Graphite flag the same issue on the same or overlapping lines in the same file, merge into one item:

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

- **Accept suggestion** — Apply the suggested change at all locations in the group
- **Fix differently** — Address the issue another way
- **Ignore** — Dismiss (will reply with reason)

### For `bot-observation`

Present: what was identified, relevant code context.

Options:

- **Address now** — Implement a fix
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

- **Agree and implement** — Make the change
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

- **Accept** — Make the change
- **Ignore** — No action

## 7. Act on decisions

### Applying code changes

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
