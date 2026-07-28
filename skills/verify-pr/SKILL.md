---
name: verify-pr
description: Verify the open GitHub pull request linked to the current branch, exercise changed UI behavior locally in Storybook or the actual application, capture and annotate desktop and mobile screenshots, publish them through 30-day signed Cloudflare Images URLs with scheduled deletion, and post a verification comment to the existing PR. Use after a PR already exists when asked to verify work, perform responsive visual QA, add screenshots, or document evidence on the current PR. Never create, commit, push, or open a PR.
---

# Verify PR

Verify the current branch's existing pull request and publish concise, annotated evidence. Do not create a PR or alter implementation code.

## Boundaries

- Never create a PR, commit, push, switch branches, or edit the PR title or description.
- Post verification as a new PR comment.
- Do not modify implementation code except for the narrow, temporary representative-data override described below. If verification fails, stop and report the failure.
- Do not claim behavior that was not exercised.
- Do not upload secrets, tokens, personal data, production customer data, or other sensitive information.
- Keep raw screenshots temporary and local. Upload only reviewed, annotated, and redacted copies.
- Use Cloudflare Images with signed delivery for every published screenshot. Make each URL expire and each uploaded image delete after 30 days. Do not commit screenshots to the repository or use GitHub's browser uploader.

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

Use Storybook when the change is an isolated component whose important states are fully represented by props.

Use the actual application when routing, navigation, authentication, permissions, server data, mutations, persistence, application layout, or multi-component interaction matters.

Use both when Storybook proves component variants and the application proves integration.

State the selected surface and why in the PR comment.

Discover and follow repository-specific commands. Do not assume that Storybook exists or invent commands. Prefer existing stories, fixtures, typed network stubs, and development scripts.

### Force representative data when necessary

When using the actual application, first try to reach the target state through realistic navigation, existing development data, fixtures, or typed network stubs.

If the correct data cannot reasonably be found or created, temporarily force representative data through hard-coded props so the changed UI can be rendered and reviewed. Treat this only as presentation verification:

- use realistic, nonsensitive values;
- make the smallest local override possible;
- do not overwrite or disturb pre-existing uncommitted work;
- never commit or push the override;
- do not claim that API integration, persistence, permissions, or end-to-end data flow was verified;
- state clearly in the PR comment that representative data was forced through props;
- remove the override immediately after capturing evidence;
- confirm that the working tree has returned exactly to its prior state before uploading or commenting.

Prefer Storybook instead when it already provides the same state more safely and accurately.

## 3. Run relevant checks

Run the smallest meaningful automated checks required by the repository and diff. Automated checks support behavioral verification but never replace it.

Exercise the changed behavior as a user:

- begin from a realistic initial state;
- perform the relevant interaction or journey;
- verify the expected result and important intermediate states;
- cover loading, empty, error, success, and permission states when the change affects them;
- verify keyboard and pointer interaction when relevant.

If any required check fails, do not upload screenshots or post a successful verification comment. Report the failure with reproducible details.

## 4. Verify responsive behavior

For every user-visible change, capture at least:

- desktop: `1440 × 900`;
- mobile: `390 × 844`.

Capture the same meaningful state at both sizes when possible. Add another viewport only when the diff affects a repository-defined breakpoint or a distinct responsive layout.

At mobile size, check:

- no horizontal overflow;
- readable wrapping and truncation;
- touch targets and reachable controls;
- fixed or sticky elements;
- dialogs, menus, keyboards, and scrolling.

At desktop size, check:

- alignment and spacing;
- content width and hierarchy;
- hover and focus behavior;
- dialogs, menus, and overlays.

Omit responsive screenshots only for genuinely non-visual work. Explain the omission in the PR comment.

## 5. Capture and annotate screenshots

Capture the verified result of an interaction, not an arbitrary initial page.

Inspect every raw screenshot before publishing it. Redact sensitive information and discard screenshots containing information that cannot be safely removed.

Annotate each screenshot to explain the changed or verified behavior:

- use at most three to five numbered callouts;
- draw high-contrast boxes or arrows around exact areas;
- keep labels outside important UI content;
- use consistent numbering and one accent color;
- describe outcomes such as `1. Submitted status appears`, not generic objects such as `1. Button`;
- split crowded evidence into multiple screenshots.

Prefer temporary browser-native overlays before capture because they preserve text quality and require no image-editor dependency. Inject uniquely named verification overlays, capture the screenshot, and remove the overlays immediately. If browser-native overlays are unavailable, use an available image annotation tool without altering the underlying application.

The PR prose must repeat the numbered callout descriptions so the evidence remains understandable and accessible.

## 6. Configure Cloudflare Images

On macOS, store the Cloudflare Images API token as a generic login Keychain item:

```text
Service:  cloudflare-images-token
Account:  <32-character Cloudflare Account ID>
Password: <Cloudflare Images API token>
```

When the item is missing, explain why it is needed and ask the user before modifying Keychain. Never ask the user to paste a token into chat.

Create it interactively with:

```bash
security add-generic-password \
  -s "cloudflare-images-token" \
  -a "YOUR_CLOUDFLARE_ACCOUNT_ID" \
  -l "Cloudflare Images Token" \
  -w
```

Keep `-w` last so macOS prompts securely. Never pass the token as a command argument, print it, log it, or use `security -A`.

Store the signing key separately in Keychain with the same safeguards. Retrieve the Account ID from the API token item's `acct` attribute, and retrieve tokens or signing keys only immediately before use. Account-owned tokens must be verified with:

```text
GET /client/v4/accounts/<ACCOUNT_ID>/tokens/verify
```

Use the user-token verification endpoint only for user-owned tokens.

On non-macOS systems, stop and give platform-appropriate secret-storage instructions. Do not silently store credentials in plaintext.

## 7. Upload evidence to Cloudflare

Upload each final annotated image:

```text
POST /client/v4/accounts/<ACCOUNT_ID>/images/v1
Authorization: Bearer <TOKEN>
multipart field: file
```

Include nonsensitive metadata when practical:

```json
{
  "repository": "owner/repository",
  "pull_request": 123,
  "commit": "full commit SHA",
  "viewport": "1440x900",
  "purpose": "desktop verification",
  "delete_after": "RFC3339 timestamp exactly 30 days after upload"
}
```

Use signed delivery only:

- Upload with `requireSignedURLs=true`. Never publish an unsigned or public fallback URL.
- Generate a signed variant URL whose expiry is exactly 30 days after upload.
- Register the Cloudflare image ID for deletion at the same 30-day deadline in the configured durable cleanup mechanism.
- Confirm the deletion schedule before posting the PR comment. URL expiry only blocks delivery; it does not delete the underlying image.
- If no signing key or durable deletion mechanism is configured, stop before uploading and report the missing prerequisite.

Prefer a scale-down variant no wider than 1600 pixels with image metadata stripped. Verify every returned URL before posting it. Stop if upload or delivery verification fails.

Record each Cloudflare image ID locally until the PR comment succeeds so failed runs can be cleaned up. After the comment succeeds, confirm the durable cleanup record contains the image ID and exact deletion timestamp.

## 8. Post the verification comment

Generate a Markdown file and post it with:

```bash
gh pr comment --body-file <verification-markdown-path>
```

Use this structure:

```markdown
## Verification evidence

Verified locally in the application because this change affects the complete user journey.

### Desktop — 1440 × 900

1. The submitted status appears in the summary.
2. The updated action remains aligned with existing controls.

![Annotated desktop verification](CLOUDFLARE_URL)

### Mobile — 390 × 844

1. The status wraps without truncation.
2. The action remains reachable without horizontal scrolling.

![Annotated mobile verification](CLOUDFLARE_URL)

### Checks

- Completed the successful user journey at both viewport sizes
- Confirmed keyboard interaction
- `repository-specific test command`

Verified against commit `FULL_COMMIT_SHA`.

Evidence links expire and uploaded images are deleted after 30 days.
```

Mention only checks that actually ran. Use descriptive image alt text. Keep the comment concise enough for reviewers to scan.

After posting, return the PR URL, comment URL when available, verified commit SHA, selected verification surface, number of uploaded screenshots, URL expiry timestamp, and scheduled image-deletion timestamp.

## Failure behavior

Do not post partial or misleading evidence. When blocked, report:

- what was attempted;
- the exact failing step;
- relevant nonsecret output;
- whether any Cloudflare images were uploaded;
- whether uploaded images were cleaned up;
- whether deletion was scheduled for any retained images;
- the smallest action required to continue.
