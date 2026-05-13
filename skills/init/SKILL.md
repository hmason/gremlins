---
description: Set up gremlins in your project — interactive wizard that generates config and prompt files
---

# Gremlins Setup Wizard

You are setting up the Gremlins system in this project. Walk the user through configuration step by step, then generate the files.

## Pre-flight Check

First, check if `gremlins/gremlins.yaml` already exists in the project:
- If it exists, tell the user: "Gremlins is already configured in this project. Want to re-run setup? This will overwrite your existing config and prompts." Wait for confirmation before proceeding.
- If it doesn't exist, proceed with setup.

## Step 1: Project Context

Ask the user:
> "What kind of project is this? Give me a sentence or two about what it does and who it's for — I'll use this to tell the gremlins what to focus on."

Take their answer and draft the `explore.context` field. Show it back to them for approval.

## Step 2: Exploration Paths

Use the Glob tool to scan the project's top-level directories (e.g., `*/` pattern). Then suggest which paths gremlins should explore:

> "I see these directories: [list]. Which should gremlins explore? I'd suggest [reasonable defaults] — sound right, or want to adjust?"

Also suggest sensible ignore patterns based on what you see (node_modules, dist, .git, vendor, __pycache__, etc.).

## Step 3: Personalities

> "Gremlins ships with three default personalities:
>
> - **The Chaos Agent** — lives for the weird and unexpected. Motto: 'What's the worst that could happen?'
> - **The Perfectionist** — finds structural flaws AND elegant fixes. Motto: 'Yes, but have you considered...'
> - **The End User** — only cares about the experience. Motto: 'But would I actually use this?'
>
> Want to use these defaults, customize them, or create your own from scratch?"

If they want to customize, walk through each personality one at a time (name, description). If from scratch, ask how many gremlins they want, then walk through each.

## Step 4: Output Mode

> "How should gremlins deliver results?
>
> - **Local files** (default) — writes to `gremlins/runs/YYYY-MM-DD/` in your project
> - **Pull request** — creates a branch and PR with the results (requires git repo)
> - **GitHub issue** — posts a summary as a GitHub issue (requires `gh` CLI)"

## Step 5: Seed Ideas

> "Want to add some starter ideas? These are rough concepts, half-baked thoughts, or 'what if' questions that gremlins can riff on. You can always add more later to `gremlins/seed-ideas.md`.
>
> Want to brainstorm a few now, or start with an empty file?"

If they want to add some, collect ideas conversationally until they say they're done.

## Step 6: Generate Files

Read the template files bundled with this skill. The templates live in the `templates/` directory next to this SKILL.md — resolve `${CLAUDE_PLUGIN_ROOT}/skills/init/templates/` if the relative path doesn't work:

- Read `templates/gremlins.yaml` as the base config
- Read `templates/orchestrator-prompt.md`
- Read `templates/gremlin-prompt-template.md`
- Read `templates/seed-ideas.md`

Modify the config template with the user's answers:
- Replace the `gremlins:` section with their chosen personalities (or keep defaults)
- Replace `explore.paths` with their chosen paths
- Replace `explore.ignore` with their chosen ignore patterns
- Replace `explore.context` with the context they approved
- Set `output.mode` to their choice
- If they chose PR mode, uncomment and set `pr_base`

Write the files to the user's project:
- `gremlins/gremlins.yaml` — modified config
- `gremlins/orchestrator-prompt.md` — copied as-is from template
- `gremlins/gremlin-prompt-template.md` — copied as-is from template
- `gremlins/seed-ideas.md` — with their seed ideas added (or the empty template)

Print a summary:

> "Gremlins is set up! Here's what I created:
>
> - `gremlins/gremlins.yaml` — your config ({N} gremlins, {output_mode} output)
> - `gremlins/orchestrator-prompt.md` — the orchestrator instructions (editable)
> - `gremlins/gremlin-prompt-template.md` — the subagent prompt template (editable)
> - `gremlins/seed-ideas.md` — your seed ideas
>
> **To run gremlins now:** `/gremlins:run`
> **To schedule recurring runs:** `/gremlins:schedule`
> **To customize behavior:** Edit the prompt files directly — they're yours."
