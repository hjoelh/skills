# Skills Repo

This repository is a `skills.sh`-installable collection of agent skills. Each installable skill lives in `skills/<slug>/`, and each skill is self-contained so agents can copy, review, and evolve them independently.

Install from GitHub:

```bash
npx skills add hjoelh/skills
```

## How Discovery Works

`skills.sh` discovers skills by looking for valid `SKILL.md` files. In this repo, all published skills live under `skills/`.

Current examples:

- `skills/pr/`
- `skills/commit/`
- `skills/grill-me/`
- `skills/make-interfaces-feel-better/`

Each skill folder should contain:

- `SKILL.md` (required)
- `agents/openai.yaml` (optional but recommended)
- any optional `assets/`, `scripts/`, or `references/` folders needed only by that skill

## Add a New Skill

1. Create a new folder at `skills/<slug>/`.
2. Add `skills/<slug>/SKILL.md` with YAML frontmatter containing:

```yaml
---
name: your-skill-name
description: Short description of when to use the skill and what it does.
---
```

3. Write concise, procedural instructions in `SKILL.md`.
4. Add `skills/<slug>/agents/openai.yaml` with matching UI metadata.
5. Test local discovery with:

```bash
npx skills add ./ --list
```

For actual install verification, run the command from outside the repo so generated agent folders do not appear in the repository working tree:

```bash
cd /tmp
npx skills add /absolute/path/to/repo --skill your-skill-name
npx skills add /absolute/path/to/repo --skill make-interfaces-feel-better
```

## Authoring Rules

- Use lowercase hyphenated folder names like `create-pr` or `deploy-preview`.
- Keep the public skill name stable once published.
- Make sure the folder name, `SKILL.md` frontmatter, and `agents/openai.yaml` all refer to the same skill identity.
- Keep `SKILL.md` concise, direct, and agent-facing.
- Put optional scripts, references, and assets inside the skill folder that uses them.
- Avoid repo-wide shared skill dependencies unless they are clearly necessary.

## Metadata Shape

`SKILL.md` is the install contract and should be the source of truth.

Minimal example:

```md
---
name: example-skill
description: Explains when to use the skill and the outcome it provides.
---

# Example Skill

Give the agent the exact workflow, guardrails, and commands it should use.
```

Recommended `agents/openai.yaml`:

```yaml
interface:
  display_name: "Example Skill"
  short_description: "Short UI description for quick scanning."
  default_prompt: "Use $example-skill to perform the task."
```

## Use the Existing Skills as Patterns

`pr` shows a concise command-oriented workflow with strong guardrails around GitHub PR creation.

`commit` shows a minimal, speed-first git workflow that stages and commits everything without asking for extra input.

If a new skill feels much more verbose or much more abstract than these two, trim it until it is mostly procedure and guardrails.

## Validation Commands

Local repo checks:

```bash
npx skills add ./ --list
cd /tmp && npx skills add /absolute/path/to/repo --skill pr
cd /tmp && npx skills add /absolute/path/to/repo --skill commit
cd /tmp && npx skills add /absolute/path/to/repo --skill make-interfaces-feel-better
```

After publishing to GitHub:

```bash
npx skills add <owner>/<repo> --list
npx skills add <owner>/<repo> --skill pr
npx skills add <owner>/<repo> --skill commit
```

## Publishing Notes

- Keep published skills in `skills/`.
- Prefer one skill per folder.
- Update this README if the repo structure or validation workflow changes.
