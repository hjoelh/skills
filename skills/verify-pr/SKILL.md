---
name: verify-pr
description: Verify the open GitHub pull request linked to the current branch, exercise changed UI behavior locally, capture high-quality desktop and mobile evidence, publish it through 30-day signed Cloudflare Images URLs, and post a verification comment to the existing PR. Use after a PR already exists when asked to verify work, perform responsive visual QA, add screenshots, or document evidence on the current PR. Never create, commit, push, or open a PR.
---

# Verify PR

Verify the current branch's existing pull request and publish concise, high-quality evidence. Do not create a PR or alter implementation code.

## Boundaries

- Never create a PR, commit, push, switch branches, or edit the PR title or description.
- Post verification as a new PR comment.
- Do not modify implementation code except for the narrow, temporary representative-data override described below. If verification fails, stop and report the failure.
- Do not claim behavior that was not exercised.
- Do not upload secrets, personal data, production customer data, or other sensitive information.
- Use Cloudflare Images with signed delivery for every published screenshot. Make each URL expire after 30 days. Do not commit evidence or use GitHub's browser uploader.
- Keep local evidence until the user has inspected and accepted it. Underlying Cloudflare uploads are retained.

## 1. Resolve the PR and verification target

Read the repository instructions before running commands.

Resolve the open PR associated with the current branch:

```bash
gh pr view --json number,url,state,headRefName,baseRefName,headRefOid,title,body
```

Stop when:

- no open PR is associated with the branch;
- the PR head branch differs from the current branch;
- local `HEAD` differs from `headRefOid`;
- uncommitted changes materially affect the behavior being verified.

Inspect the PR body, linked issue when available, and the complete branch diff against the PR base. Derive concrete acceptance criteria from the user-visible changes. Record the current commit SHA for the final comment.

## 2. Choose the verification surface

Use the surface that can faithfully exercise the behavior:

- Use Storybook for an isolated component whose important states are fully represented by props.
- Use the application when routing, authentication, permissions, server data, persistence, layout, or multi-component interaction matters.
- Use both when component variants and application integration need separate proof.

State the selected surface and why in the PR comment. Discover and follow repository-specific commands; do not invent them.

### Force representative data when necessary

First try realistic navigation, existing development data, fixtures, or typed network stubs. If the state cannot reasonably be reached, temporarily force realistic, nonsensitive props:

- make the smallest local override possible;
- do not disturb pre-existing work, commit, or push the override;
- treat the result only as presentation verification;
- disclose the override and do not claim integration coverage;
- remove it immediately after capture and restore the exact prior working-tree state.

Prefer an existing Storybook story when it represents the same state more safely.

## 3. Exercise the behavior

Run the smallest meaningful automated checks required by the repository and diff, then exercise the changed behavior as a user. Cover the states and interactions affected by the change, including keyboard and pointer behavior when relevant.

If a required check fails, do not upload screenshots or post a successful verification comment. Report the failure with reproducible details.

## 4. Verify responsive behavior

For every visual change, verify at least:

- desktop: `1440 × 900` CSS pixels;
- mobile: `390 × 844` CSS pixels.

Use the same meaningful state at both sizes when possible. Add another viewport only for a relevant breakpoint or distinct layout. On mobile, check overflow, wrapping, touch targets, fixed elements, dialogs, and scrolling. On desktop, check alignment, hierarchy, hover, focus, and overlays.

Omit responsive evidence only for genuinely non-visual work and explain the omission.

## 5. Produce high-quality evidence

Verification, browser control, and capture may use different tools. Choose any safe capture and annotation approach that meets the quality requirements; do not assume a screenshot is suitable because a particular tool produced it.

For every final image:

- capture the verified result, not an arbitrary initial state;
- make the image sharp, high-resolution, and free of visible compression artifacts;
- record the CSS viewport separately from the output pixel dimensions;
- ensure text, thin borders, and callouts remain clear at 100% zoom and at the size GitHub renders;
- ensure the changed UI is large enough to review at GitHub's rendered size;
- include a tight detail crop when the full responsive view makes the changed region too small;
- redact sensitive information and discard evidence that cannot be made safe.

Keep annotations concise: use no more than three to five numbered callouts, avoid covering important content, and repeat their descriptions in the PR prose.

## 6. Configure Cloudflare Images

Use two separate generic login Keychain items on macOS:

```text
Service:  cloudflare-images-token
Account:  <32-character Cloudflare Account ID>
Password: <Cloudflare Images API token>

Service:  cloudflare-images-signing-key
Account:  default
Password: <Cloudflare Images signing key>
```

Use `cloudflare-images-token` only for Cloudflare API authentication. Use `cloudflare-images-signing-key` only to sign delivery URLs. Retrieve each secret immediately before use; never print, log, pass it as a command argument, or write it to the local evidence record.

When either item is missing, explain why it is needed and ask before modifying Keychain. Never ask the user to paste a secret into chat. Create missing items interactively with `security add-generic-password ... -w`, keeping `-w` last, and never use `security -A`.

On non-macOS systems, stop and give platform-appropriate secret-storage instructions. Do not store credentials in plaintext.

## 7. Upload and validate evidence

Use the existing predefined `public` variant. Do not create or modify Cloudflare variants. Before upload, require `public` to:

- use `fit: scale-down`;
- use `metadata: none`;
- have width and height bounds at least as large as the source image;
- set `neverRequireSignedURLs: false`.

For each final image:

- validate the API token and `public` variant;
- upload with `requireSignedURLs=true`;
- sign the `public` delivery URL for exactly 30 days with `cloudflare-images-signing-key`;
- fetch and visually inspect the signed delivery before posting;
- confirm the delivered image remains high quality, readable, and appropriately sized;
- retain a local, secret-free record of the image ID, signed URL, expiry, viewport, dimensions, and PR metadata.

If the delivered image is blurry, compressed, unexpectedly resized, or otherwise hard to review, do not post it. Diagnose the affected stage and recapture or re-upload until the delivered evidence is high quality.

When replacing evidence, re-upload it and generate a new valid signed URL. Do not append unsigned cache-busting parameters.

## 8. Post the verification comment

Generate a Markdown file and post it with:

```bash
gh pr comment --body-file <verification-markdown-path>
```

Use this structure:

```markdown
## Verification evidence

Verified locally in the application because this change affects the complete user journey.

### Desktop — 1440 × 900 CSS viewport

Delivered image: 1440 × 900 pixels

1. The submitted status appears in the summary.
2. The updated action remains aligned with existing controls.

![Annotated desktop verification](CLOUDFLARE_URL)

### Mobile — 390 × 844 CSS viewport

Delivered image: 390 × 844 pixels

1. The status wraps without truncation.
2. The action remains reachable without horizontal scrolling.

![Annotated mobile verification](CLOUDFLARE_URL)

### Checks

- Completed the successful user journey at both viewport sizes
- Confirmed keyboard interaction
- `repository-specific test command`

Verified against commit `FULL_COMMIT_SHA`.

Evidence links expire after 30 days. Underlying Cloudflare uploads are retained.
```

Mention only checks that actually ran. Disclose representative props or data and distinguish presentation verification from integration coverage. Use descriptive image alt text.

After posting, add the returned comment URL to the local evidence record. Return the PR URL, comment URL, verified commit SHA, selected surface, evidence count, expiry timestamp, and local evidence directory. Keep that directory until the user accepts the evidence, then remove only the local artifacts.

## Failure behavior

Do not post partial or misleading evidence. Report:

- what was attempted and the exact failing step;
- relevant nonsecret output;
- whether any Cloudflare images were uploaded;
- whether an incomplete upload was cleaned up;
- the local evidence directory and record when present;
- the smallest action required to continue.
