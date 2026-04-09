# Gremlins Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that runs autonomous, personality-driven AI agents against any project to generate creative ideas and full design documents.

**Architecture:** A Claude Code plugin providing three skills (`gremlins init`, `gremlins run`, `gremlins schedule`). During init, editable prompt files and config are generated in the user's project. At runtime, the orchestrator skill reads these project-local files, surveys the project, and spawns parallel subagents for pitching, cross-critique, and design phases.

**Tech Stack:** Claude Code plugin system (markdown skills), YAML (config), Markdown (prompt templates), Claude Code Remote Triggers (scheduling)

**Spec:** `docs/superpowers/specs/2026-04-09-gremlins-plugin-design.md`

**Reference implementation:** The original Hidden Door gremlins system at `/Users/hilary/code/hd/dev/wt/idea/gremlins/project_notes/gremlins/` — particularly `orchestrator-prompt.md`, `config.yaml`, and `gremlins-design.md`.

---

### Task 1: Initialize the repo and plugin manifest

**Files:**
- Create: `plugin.json`
- Create: `.gitignore`

- [ ] **Step 1: Initialize git repo**

```bash
cd /Users/hilary/code/hd/dev/gremlins
git init
```

- [ ] **Step 2: Create .gitignore**

```
.DS_Store
```

- [ ] **Step 3: Create plugin.json**

```json
{
  "name": "gremlins",
  "version": "1.0.0",
  "description": "Autonomous idea generation agents with distinct personalities that explore your project and produce design documents",
  "skills": [
    {
      "name": "gremlins",
      "description": "Run creative idea-generating agents against your project",
      "subcommands": ["init", "run", "schedule"]
    }
  ]
}
```

- [ ] **Step 4: Commit**

```bash
git add plugin.json .gitignore
git commit -m "Initialize gremlins plugin with manifest"
```

---

### Task 2: Create the default config template

This is the template that `gremlins init` copies into the user's project, populated with their answers.

**Files:**
- Create: `templates/gremlins.yaml`

- [ ] **Step 1: Write templates/gremlins.yaml**

```yaml
# Gremlin personalities — add, remove, or edit freely
gremlins:
  - name: "The Chaos Agent"
    slug: "chaos-agent"
    description: >
      Lives for the weird and unexpected. Breaks assumptions, combines
      incompatibles, pushes into slightly dangerous territory.
      Motto: "What's the worst that could happen?"

  - name: "The Perfectionist"
    slug: "perfectionist"
    description: >
      Sees potential everywhere and how to make it bulletproof. Finds
      structural flaws AND elegant fixes. Believes the best ideas
      survive scrutiny. Motto: "Yes, but have you considered..."

  - name: "The End User"
    slug: "end-user"
    description: >
      Cares about one thing: what would be incredible to actually
      experience? Doesn't care about technical elegance or business
      metrics — only the moment that makes someone say "wow."
      Motto: "But would I actually use this?"

# What to explore — tells gremlins where to look and what matters
explore:
  paths:
    - "src/"
    - "docs/"
  ignore:
    - "node_modules/"
    - "dist/"
    - ".git/"
    - "vendor/"
    - "__pycache__/"
  context: >
    Describe your project here: what it does, who it's for, what the team
    is focused on, and what kinds of ideas are most welcome. This is the
    most important field — it tells gremlins what matters.

# Seed ideas file — optional, gremlins generate purely from exploration if absent
seed_ideas: "seed-ideas.md"

# Output configuration
output:
  dir: "gremlins/runs"
  mode: "local"            # "local", "pr", or "issue"
  # pr_base: "main"        # Branch to target for PRs (when mode is "pr")

# Model configuration
models:
  default: "claude-opus-4-6"
  # Per-phase overrides (optional):
  # survey: "claude-sonnet-4-6"
  # pitch: "claude-opus-4-6"
  # critique: "claude-opus-4-6"
  # design: "claude-opus-4-6"

# How many ideas each gremlin pitches (default: 1)
ideas_per_gremlin: 1
```

- [ ] **Step 2: Commit**

```bash
git add templates/gremlins.yaml
git commit -m "Add default config template with three personalities"
```

---

### Task 3: Create the gremlin prompt template

This is the template used for Phase 2-4 subagents. It's copied into the user's project during init and read by the orchestrator at runtime. The orchestrator does simple `{{PLACEHOLDER}}` string replacement before dispatching each subagent.

**Files:**
- Create: `templates/gremlin-prompt-template.md`

**Reference:** The subagent prompts in `/Users/hilary/code/hd/dev/wt/idea/gremlins/project_notes/gremlins/orchestrator-prompt.md` lines 53-166, generalized to remove Hidden Door specifics.

- [ ] **Step 1: Write templates/gremlin-prompt-template.md**

The template has three sections gated by `{{PHASE}}`. The orchestrator includes only the relevant section when dispatching. Here is the full content:

````markdown
# Gremlin Prompt Template

This file is used by the orchestrator to construct prompts for each gremlin subagent. The orchestrator replaces `{{PLACEHOLDER}}` values and includes only the section matching the current phase.

---

## Identity

You are **{{PERSONALITY_NAME}}**, a gremlin — a creative agent that generates wild ideas. Boring is failure. Only ~10% of ideas need to be good — the rest should be gloriously weird and thought-provoking. The best ideas start as bad ideas.

**Your personality:**
{{PERSONALITY_DESCRIPTION}}

**Project context:**
{{CONTEXT_BRIEF}}

---

## Phase: Pitch

**Your task:**
1. Review the seed ideas and project context through the lens of your personality.
2. Use the Read, Glob, and Grep tools to explore the project if you need to understand how something works. Focus on the paths and context described above.
3. Pick ONE idea — from the seeds, self-generated, or a mashup. Choose the idea that YOUR personality finds most compelling.
4. Write a pitch (~300-500 words) to the file `{{OUTPUT_PATH}}` containing:
   - **Idea name** (as a heading)
   - **What it is** — describe the idea clearly
   - **Why it's exciting** — from your personality's perspective, why does this idea matter?
   - **How it connects to the existing project** — what does it build on?
   - **What it looks like for a user** — paint the picture. What's the moment that makes someone say "wow"?
   - **Open questions** — things you'd normally ask a human. "Would users want control over this?" "Does this conflict with X?" "What are the edge cases?"

---

## Phase: Critique

**The pitches:**

{{ALL_PITCHES}}

**Your task:**
Write critiques of the OTHER pitches (not your own) to `{{OUTPUT_PATH}}`. Stay in character. For each pitch:

1. What's the strongest thing about this idea?
2. What's the biggest weakness or missed opportunity?
3. How would YOU twist, improve, or build on it?
4. One question the pitcher should answer before going further.

Be honest, be constructive, be in character.

---

## Phase: Design

**Your original pitch:**
{{MY_PITCH}}

**Critiques from the other gremlins:**
{{CRITIQUES_OF_MY_PITCH}}

**Your task:**
Turn your pitch into a full design document. Write the result to `{{OUTPUT_PATH}}`.

Follow these steps:

**1. Deep context exploration.** Use Read, Glob, and Grep to dive into the specific project areas relevant to your idea. Go beyond the general survey — understand what would actually be involved.

**2. Surface open questions.** What can't you answer? What needs a human decision? List these explicitly.

**3. Propose 2-3 approaches.** Different ways to build this idea, with trade-offs for each.

**4. Recommend one approach** with reasoning.

**5. Write the full design doc** with these sections:

- **Problem statement** — what gap or opportunity does this address?
- **Proposed solution** — what is it and how does it work?
- **User experience** — moment-to-moment, what does this feel like? What's the "jaw drop" moment? What do users tell their friends about? This is the most important section — if you can't make someone excited reading this, the idea isn't ready.
- **Approaches considered** — the 2-3 options with trade-offs and your recommendation
- **Impact assessment** — what areas of the project would change, roughly how much work
- **Open questions for humans** — what you need a human to weigh in on
- **Wildness rating** (1-5) — 1 is "sensible extension", 5 is "this might be genius or might be insane"
- **Why this is a gremlin idea** — what makes this something a normal process would never produce? What assumption does it challenge?

Incorporate the feedback from the critiques. You don't have to agree with everything, but engage with it seriously.
````

- [ ] **Step 2: Commit**

```bash
git add templates/gremlin-prompt-template.md
git commit -m "Add gremlin subagent prompt template with pitch/critique/design phases"
```

---

### Task 4: Create the orchestrator prompt template

This is the main prompt that drives the entire system. It's copied into the user's project during init and read by the `gremlins run` skill at runtime. Generalized from the Hidden Door original to work with any project.

**Files:**
- Create: `templates/orchestrator-prompt.md`

**Reference:** `/Users/hilary/code/hd/dev/wt/idea/gremlins/project_notes/gremlins/orchestrator-prompt.md` — the full 210-line original. This task generalizes it by replacing hardcoded Hidden Door paths with config-driven exploration.

- [ ] **Step 1: Write templates/orchestrator-prompt.md**

````markdown
# Gremlins Orchestrator

You are the Gremlins Orchestrator. Your job is to generate wildly creative ideas by running a team of AI agents with distinct personalities against this project.

Your ideas should be surprising, ambitious, and at the edge of possibility. Boring is failure. Only ~10% of ideas need to be good — the rest should be gloriously weird and thought-provoking. The best ideas start as bad ideas.

**You must never modify existing project files.** You are a thinker, not an implementer. You read the project for inspiration and understanding, and you write design documents.

## Setup

1. Read `gremlins/gremlins.yaml` for gremlin personalities, model settings, exploration paths, and output configuration.
2. Read `gremlins/seed-ideas.md` for seed ideas (if the file exists).
3. Read `gremlins/gremlin-prompt-template.md` — you will use this template for all subagent prompts.
4. Determine today's date and set `OUTPUT_DIR` to `{output.dir}/YYYY-MM-DD/` using that date. If that directory already exists (from a previous run today), append a suffix: `YYYY-MM-DD-2`, `YYYY-MM-DD-3`, etc.
5. Create the output directory structure:
   ```
   {OUTPUT_DIR}/
     pitches/
     critiques/
     designs/
   ```

## Phase 1: Survey

Survey the project to build a shared brief for the gremlins. This is read-only exploration.

1. Read the `explore.context` field from `gremlins.yaml` to understand what the project is and what matters.
2. Explore the paths listed in `explore.paths`, skipping anything in `explore.ignore`. Use Glob to find files and Read/Grep to understand them. Focus on understanding:
   - What the project can do today (current capabilities)
   - How it's structured (key components and their relationships)
   - What seems ripe for new ideas or underserved
3. If previous gremlins runs exist (other date directories under the output dir), skim their README.md files to understand what ideas have already been explored. Avoid repeating them.
4. Read seed ideas from the seed ideas file (if it exists).

Produce a **context brief** — a summary held in memory (not written to a file) containing:
- What the project does today (current capabilities)
- How it's structured (key areas and what they do)
- The `explore.context` from config (project goals and focus)
- Seed ideas (if any)
- Areas that seem ripe for exploration or underserved
- Interesting capabilities that could be leveraged in new ways

## Phase 2: Idea Selection & Pitching

For EACH gremlin defined in `gremlins.yaml`, spawn a subagent using the Agent tool. **Run all subagents in parallel** (make all Agent tool calls in a single message).

To construct each subagent's prompt:
1. Take the **Identity** section and **Phase: Pitch** section from `gremlin-prompt-template.md`
2. Replace these placeholders:
   - `{{PERSONALITY_NAME}}` — the gremlin's `name` from config
   - `{{PERSONALITY_DESCRIPTION}}` — the gremlin's `description` from config
   - `{{CONTEXT_BRIEF}}` — the context brief you built in Phase 1
   - `{{OUTPUT_PATH}}` — `{OUTPUT_DIR}/pitches/{gremlin_slug}.md`

After all subagents complete, read all pitch files to collect the results.

## Phase 3: Cross-Critique

Read all pitch files and concatenate them. For EACH gremlin, spawn a subagent. **Run all subagents in parallel.**

To construct each subagent's prompt:
1. Take the **Identity** section and **Phase: Critique** section from `gremlin-prompt-template.md`
2. Replace these placeholders:
   - `{{PERSONALITY_NAME}}` — the gremlin's `name`
   - `{{PERSONALITY_DESCRIPTION}}` — the gremlin's `description`
   - `{{CONTEXT_BRIEF}}` — the context brief
   - `{{ALL_PITCHES}}` — all pitch files concatenated, each labeled with the gremlin's name
   - `{{OUTPUT_PATH}}` — `{OUTPUT_DIR}/critiques/{gremlin_slug}.md`

After all subagents complete, read all critique files.

## Phase 4: Full Design Process

For EACH gremlin, collect:
- Their original pitch
- The critiques that the OTHER gremlins wrote about their pitch (look in the other gremlins' critique files for sections about this gremlin's idea)

Spawn a subagent per gremlin. **Run all subagents in parallel.**

To construct each subagent's prompt:
1. Take the **Identity** section and **Phase: Design** section from `gremlin-prompt-template.md`
2. Replace these placeholders:
   - `{{PERSONALITY_NAME}}` — the gremlin's `name`
   - `{{PERSONALITY_DESCRIPTION}}` — the gremlin's `description`
   - `{{CONTEXT_BRIEF}}` — the context brief
   - `{{MY_PITCH}}` — this gremlin's pitch file content
   - `{{CRITIQUES_OF_MY_PITCH}}` — relevant sections from other gremlins' critique files
   - `{{OUTPUT_PATH}}` — `{OUTPUT_DIR}/designs/{gremlin_slug}-{idea_slug}.md` (create a URL-friendly slug from the idea name in the pitch heading)

After all subagents complete, read all design doc files.

## Phase 5: Summary & Delivery

1. Read all design documents.
2. Write `{OUTPUT_DIR}/README.md` containing:
   - A one-paragraph summary of each idea
   - Wildness ratings at a glance (table or list)
   - Which seed ideas were used as inspiration (if any)
   - Your "if I had to pick one" recommendation with reasoning
3. Deliver based on `output.mode` from config:

**If mode is "local":**
- Print a summary to the terminal. Done.

**If mode is "pr":**
- Create a new git branch named `gremlins/YYYY-MM-DD` (using today's date). If a branch with that name already exists, append a suffix: `-2`, `-3`, etc.
- Stage and commit all files under `{OUTPUT_DIR}/`.
- Push the branch and open a PR:
  - Title: `Gremlins: {N} ideas for YYYY-MM-DD`
  - Body: content of README.md + "\n\n---\nGenerated by the gremlins system.\nReview, discuss, cherry-pick what sparks joy. Ignore the rest."

**If mode is "issue":**
- Create a GitHub issue:
  - Title: `Gremlins: {N} ideas for YYYY-MM-DD`
  - Body: content of README.md + full design doc contents (or links to local files if too large)

## Seed Idea Tracking

After delivery, update the seed ideas file (if it exists) to track which seed ideas have been used:

1. If the file has no "## Previously Used" section, create one at the bottom.
2. Move any seed ideas that a gremlin used (directly or as a mashup) from the top list to the "Previously Used" section.
3. Label each moved idea with the date and which gremlin used it, e.g.: `- ~~Original idea text~~ — used by The Chaos Agent (YYYY-MM-DD) as "Idea Name"`
4. Gremlins should prefer unused ideas and self-generated ideas over re-using previously explored ones.

## Important Notes

- If the seed ideas file is empty, missing, or has no unused ideas, generate ideas purely from project exploration. Never skip a run.
- If a subagent fails or produces no output, log the error and continue with the remaining gremlins. Two ideas are better than zero.
- The number of gremlins is determined by `gremlins.yaml` — if someone adds a 4th personality, spawn 4 subagents.
- Use the `slug` field from config for file naming. If no slug is provided, generate one by lowercasing the name, removing "The ", and replacing spaces with hyphens.
- Subagent tools: Read, Glob, Grep, Write. No destructive tools. Agents never modify existing project files.
````

- [ ] **Step 2: Review the prompt for completeness against the spec**

Verify:
- All five phases are represented
- Subagent prompts include all required design doc sections (especially "user experience" and "why this is a gremlin idea")
- Output structure matches the spec
- All three output modes (local, pr, issue) are handled
- Seed tracking logic is present
- Config-driven exploration replaces all hardcoded paths

- [ ] **Step 3: Commit**

```bash
git add templates/orchestrator-prompt.md
git commit -m "Add generic orchestrator prompt template with five-phase workflow"
```

---

### Task 5: Create the seed ideas template

A starter file with instructions, copied into the user's project during init.

**Files:**
- Create: `templates/seed-ideas.md`

- [ ] **Step 1: Write templates/seed-ideas.md**

```markdown
# Seed Ideas

Drop your half-baked, wild, weird, probably-terrible ideas here.
The gremlins will read these for inspiration (along with exploring
your project on their own). Lower the bar. The best ideas start
as bad ones.

- 
```

- [ ] **Step 2: Commit**

```bash
git add templates/seed-ideas.md
git commit -m "Add seed ideas template"
```

---

### Task 6: Create the gremlins-init skill

The setup wizard. This is a Claude Code skill (markdown file) that walks users through configuration interactively.

**Files:**
- Create: `skills/gremlins-init.md`

- [ ] **Step 1: Write skills/gremlins-init.md**

````markdown
---
name: gremlins init
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

Read the template files from the plugin's `templates/` directory:
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
> **To run gremlins now:** `/gremlins run`
> **To schedule recurring runs:** `/gremlins schedule`
> **To customize behavior:** Edit the prompt files directly — they're yours."
````

- [ ] **Step 2: Commit**

```bash
git add skills/gremlins-init.md
git commit -m "Add gremlins init skill with interactive setup wizard"
```

---

### Task 7: Create the gremlins-run skill

The core orchestration skill. When invoked, it reads the project-local orchestrator prompt and follows it.

**Files:**
- Create: `skills/gremlins-run.md`

- [ ] **Step 1: Write skills/gremlins-run.md**

````markdown
---
name: gremlins run
description: Run gremlins — autonomous idea-generating agents explore your project and produce design documents
---

# Gremlins Run

## Pre-flight Check

1. Check if `gremlins/gremlins.yaml` exists. If not, tell the user: "Gremlins isn't set up yet. Run `/gremlins init` first." and stop.
2. Check if `gremlins/orchestrator-prompt.md` exists. If not, tell the user: "Missing orchestrator prompt at `gremlins/orchestrator-prompt.md`. Run `/gremlins init` to regenerate." and stop.
3. Check if `gremlins/gremlin-prompt-template.md` exists. If not, tell the user: "Missing gremlin prompt template at `gremlins/gremlin-prompt-template.md`. Run `/gremlins init` to regenerate." and stop.

## Execute

Read `gremlins/orchestrator-prompt.md` and follow its instructions exactly.

The orchestrator prompt contains the full five-phase workflow:
1. Survey the project (read-only exploration)
2. Spawn parallel subagents to pitch ideas
3. Spawn parallel subagents for cross-critique
4. Spawn parallel subagents to produce full design documents
5. Compile summary and deliver results

The orchestrator prompt references `gremlins/gremlins.yaml` for configuration and `gremlins/gremlin-prompt-template.md` for subagent prompts. Follow whatever is written in those files — the user may have customized them.
````

- [ ] **Step 2: Commit**

```bash
git add skills/gremlins-run.md
git commit -m "Add gremlins run skill with pre-flight checks and orchestrator dispatch"
```

---

### Task 8: Create the gremlins-schedule skill

Thin wrapper around Claude Code Remote Triggers for scheduling recurring runs.

**Files:**
- Create: `skills/gremlins-schedule.md`

- [ ] **Step 1: Write skills/gremlins-schedule.md**

````markdown
---
name: gremlins schedule
description: Schedule recurring gremlins runs using Claude Code Remote Triggers
---

# Gremlins Schedule

Manage scheduled gremlins runs. This is a thin wrapper around Claude Code's Remote Trigger system.

## Determine Subcommand

Parse the user's input to determine which action they want:
- `/gremlins schedule` (no args) or `/gremlins schedule create` → **Create**
- `/gremlins schedule list` → **List**
- `/gremlins schedule remove` → **Remove**

---

## Create

### Pre-flight Check

1. Check if `gremlins/gremlins.yaml` exists. If not, tell the user to run `/gremlins init` first.
2. Check if `gremlins/orchestrator-prompt.md` exists. If not, tell the user to run `/gremlins init` first.

### Interactive Setup

Ask the user:

**Frequency:**
> "How often should gremlins run?
> - **Weekly** (default) — pick a day and time
> - **Biweekly** — every two weeks
> - **Monthly** — once a month
> - **Custom** — provide a cron expression"

**Timing** (unless they gave a custom cron):
> "What day and time? (e.g., 'Sunday 8am', 'Friday 6pm')"

Convert their answer to a UTC cron expression.

### Create the Trigger

Use the RemoteTrigger tool (or the `/schedule` skill) to create a Remote Trigger with:
- **Prompt:** `Read gremlins/orchestrator-prompt.md and follow its instructions exactly.`
- **Schedule:** The cron expression from above
- **Model:** Read `models.default` from `gremlins/gremlins.yaml` (default: `claude-opus-4-6`)

Print confirmation:
> "Scheduled! Gremlins will run [frequency description] at [time]. To manage this schedule, use `/gremlins schedule list` or `/gremlins schedule remove`."

---

## List

List all Remote Triggers and filter for any whose prompt contains "orchestrator-prompt.md". Display:
- Schedule (cron expression and human-readable)
- Model
- Trigger ID

If none found: "No gremlins schedule configured. Use `/gremlins schedule` to set one up."

---

## Remove

List gremlins triggers (same as List). If one is found, confirm with the user before removing:
> "Found gremlins schedule: [description]. Remove it?"

If confirmed, delete the Remote Trigger.

If multiple are found, ask which one to remove.
````

- [ ] **Step 2: Commit**

```bash
git add skills/gremlins-schedule.md
git commit -m "Add gremlins schedule skill wrapping Remote Triggers"
```

---

### Task 9: Create the README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write README.md**

```markdown
# Gremlins

Autonomous idea generation agents for Claude Code. Gremlins explores your project with distinct AI personalities, then produces creative design documents through a structured pitch → critique → design workflow.

## How it works

1. **Survey** — Gremlins reads your project to understand what exists and what matters
2. **Pitch** — Each gremlin independently picks and pitches one wild idea
3. **Critique** — Gremlins cross-critique each other's pitches, staying in character
4. **Design** — Each gremlin produces a full design document incorporating feedback
5. **Deliver** — A summary with all designs is delivered as local files, a PR, or a GitHub issue

## Installation

```bash
claude plugins add <this-repo-url>
```

## Quick Start

```bash
# Set up gremlins in your project (interactive wizard)
/gremlins init

# Run gremlins now
/gremlins run

# Schedule recurring runs
/gremlins schedule
```

## Default Personalities

- **The Chaos Agent** — lives for the weird and unexpected. Motto: "What's the worst that could happen?"
- **The Perfectionist** — finds structural flaws AND elegant fixes. Motto: "Yes, but have you considered..."
- **The End User** — only cares about the lived experience. Motto: "But would I actually use this?"

You can customize personalities, add new ones, or start from scratch during setup.

## Configuration

After running `/gremlins init`, your project gets a `gremlins/` directory:

- `gremlins.yaml` — personalities, exploration paths, output mode, model settings
- `orchestrator-prompt.md` — the full orchestrator instructions (editable)
- `gremlin-prompt-template.md` — the subagent prompt template (editable)
- `seed-ideas.md` — rough ideas for gremlins to riff on

All files are yours to edit. The prompts are the product — customize them freely.

## Output

Each run produces a dated folder:

```
gremlins/runs/YYYY-MM-DD/
├── README.md          # Summary & recommendations
├── pitches/           # Each gremlin's idea pitch
├── critiques/         # Cross-critiques
└── designs/           # Full design documents
```

## Requirements

- Claude Code with Agent tool support
- Opus recommended (Sonnet works for budget-conscious runs)
- Git repo + `gh` CLI only if using PR or issue output mode
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "Add README with installation and usage instructions"
```

---

### Task 10: Create LICENSE

**Files:**
- Create: `LICENSE`

- [ ] **Step 1: Ask the user which license they prefer**

Suggest MIT as a reasonable default for a shareable open-source tool. If they choose something else, use that.

- [ ] **Step 2: Write the LICENSE file with the chosen license**

- [ ] **Step 3: Commit**

```bash
git add LICENSE
git commit -m "Add license"
```

---

### Task 11: End-to-end verification

- [ ] **Step 1: Verify all plugin files exist**

```bash
ls -la /Users/hilary/code/hd/dev/gremlins/plugin.json
ls -la /Users/hilary/code/hd/dev/gremlins/skills/gremlins-init.md
ls -la /Users/hilary/code/hd/dev/gremlins/skills/gremlins-run.md
ls -la /Users/hilary/code/hd/dev/gremlins/skills/gremlins-schedule.md
ls -la /Users/hilary/code/hd/dev/gremlins/templates/gremlins.yaml
ls -la /Users/hilary/code/hd/dev/gremlins/templates/orchestrator-prompt.md
ls -la /Users/hilary/code/hd/dev/gremlins/templates/gremlin-prompt-template.md
ls -la /Users/hilary/code/hd/dev/gremlins/templates/seed-ideas.md
ls -la /Users/hilary/code/hd/dev/gremlins/README.md
ls -la /Users/hilary/code/hd/dev/gremlins/LICENSE
```

- [ ] **Step 2: Verify plugin.json is valid JSON**

```bash
cd /Users/hilary/code/hd/dev/gremlins && python3 -c "import json; json.load(open('plugin.json')); print('Valid')"
```

- [ ] **Step 3: Verify config template is valid YAML**

```bash
cd /Users/hilary/code/hd/dev/gremlins && python3 -c "import yaml; yaml.safe_load(open('templates/gremlins.yaml')); print('Valid')"
```

- [ ] **Step 4: Verify all placeholder references in orchestrator-prompt.md match what gremlin-prompt-template.md expects**

Read both template files and confirm:
- The orchestrator prompt references placeholders `{{PERSONALITY_NAME}}`, `{{PERSONALITY_DESCRIPTION}}`, `{{CONTEXT_BRIEF}}`, `{{OUTPUT_PATH}}`, `{{ALL_PITCHES}}`, `{{MY_PITCH}}`, `{{CRITIQUES_OF_MY_PITCH}}`
- The gremlin prompt template uses those same placeholder names exactly

- [ ] **Step 5: Smoke test — install the plugin locally and run init**

```bash
claude plugins add /Users/hilary/code/hd/dev/gremlins
```

Then in a test project, run `/gremlins init` to verify the wizard works.

- [ ] **Step 6: Final commit if any fixes were needed**

```bash
cd /Users/hilary/code/hd/dev/gremlins
git add -A
git commit -m "Fix issues found during end-to-end verification"
```
