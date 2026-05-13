# Gremlins: Claude Code Plugin Design

A Claude Code plugin that runs autonomous, personality-driven AI agents against any project to generate creative ideas and full design documents.

## Background

Gremlins was originally built for Hidden Door's "Untitled" monorepo — three AI agents with distinct creative personalities that explore the codebase weekly, pitch ideas, cross-critique each other, and produce implementable design documents. This spec describes extracting that system into a generic, shareable Claude Code plugin.

## Distribution Model

Claude Code plugin, installed via `claude plugins add <github-url>`. Provides three slash commands: `/gremlins init`, `/gremlins run`, `/gremlins schedule`.

## Architecture: Skill + Project-Local Prompt Files

The plugin provides the orchestration skills. During `gremlins init`, it generates editable prompt files and configuration in the user's project. At runtime, the skill reads these project-local files rather than using baked-in templates. This means:

- Users can see and tweak exactly what the agents do
- Prompts serve as documentation
- Power users can deeply customize without forking the plugin
- Plugin upgrades don't silently change behavior

---

## Plugin Repo Structure

```
gremlins/
├── plugin.json                    # Plugin manifest
├── skills/
│   ├── gremlins-run.md            # Main orchestrator skill (/gremlins run)
│   ├── gremlins-init.md           # Setup wizard skill (/gremlins init)
│   └── gremlins-schedule.md       # Schedule wrapper skill (/gremlins schedule)
├── templates/                     # Copied into user's project during init
│   ├── orchestrator-prompt.md
│   ├── gremlin-prompt-template.md
│   └── gremlins.yaml
├── README.md
└── LICENSE
```

## User's Project (After Init)

```
gremlins/
├── gremlins.yaml                  # Config: personalities, paths, output mode
├── orchestrator-prompt.md         # Editable orchestrator instructions
├── gremlin-prompt-template.md     # Editable subagent prompt template
├── seed-ideas.md                  # Optional seed ideas
└── runs/                          # Output from runs
    └── YYYY-MM-DD/
        ├── README.md              # Summary & recommendations
        ├── pitches/
        │   └── {slug}.md
        ├── critiques/
        │   └── {slug}.md
        └── designs/
            └── {slug}-{idea-slug}.md
```

---

## Configuration (`gremlins.yaml`)

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
  context: >
    Free-text description of the project, what matters, what the team
    is focused on, and what kinds of ideas are most welcome. This is
    the key differentiator — it makes gremlins useful for non-codebase
    contexts too (business strategy, content planning, etc.).

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

### Key config decisions

- **`explore.context`** is the main lever for making gremlins useful across domains. A content strategist sets paths to their docs folder and writes context about their audience. A startup founder points at their pitch deck and business plan.
- **`explore.paths` + `explore.ignore`** handle the mechanical "where to look" problem. The setup wizard suggests these based on the project structure.
- **Model overrides per phase** let users save tokens on the survey phase (Sonnet is fine for reading) while keeping Opus for creative work.
- **`output.mode`** defaults to `local` (safest). PR and issue modes require git repo and `gh` CLI respectively.

---

## Setup Wizard (`gremlins init`)

Interactive, conversational setup — one question per step:

1. **Project context** — "What kind of project is this?" Drafts `explore.context`.
2. **Exploration paths** — Scans directory structure, suggests paths. "I see `src/`, `docs/`, `tests/`. Which should gremlins explore?"
3. **Personalities** — "Use the three defaults (Chaos Agent, Perfectionist, End User), customize them, or start from scratch?"
4. **Output mode** — "How should gremlins deliver results? Local files (default), PR, or GitHub issue?"
5. **Seed ideas** — "Want to add some starter ideas now, or start with an empty file?"
6. **Generate files** — Writes config, prompts, and seed file. Prints summary.

---

## Idea Types: Features vs. Content

Pitches can be **features** (new capabilities, system changes) or **content** (a specific new instance of an existing data type — a new template, a new entry in a registry, a new value in an enum, a new entry in a content list). Both are first-class outputs. For projects with obvious content surfaces — registries, template lists, data files — a single juicy content artifact is a real shippable thing, not a lesser idea than a feature. The orchestrator surfaces content surfaces during the survey; gremlins choose freely.

## Orchestration (`gremlins run`)

### Setup
- Reads config, seed ideas, and the gremlin prompt template
- Creates output directory `runs/YYYY-MM-DD/{pitches,critiques,designs}/`
- **If `output.mode` is `pr`, creates and checks out the branch `gremlins/YYYY-MM-DD` upfront** so each phase can commit incrementally and a partial run still produces a reviewable PR

### Phase 1: Survey (Orchestrator, read-only)
- Reads `gremlins.yaml` for config
- Reads `seed-ideas.md` if present
- Explores paths in `explore.paths`, skipping `explore.ignore`
- Reads `explore.context` to understand what matters
- Notes content surfaces (registries, template lists, data files) where a "new entry" is a meaningful unit of work
- Checks previous runs in `output.dir` to avoid repeating ideas
- Produces an in-memory context brief (≤ 500 words)

### Phase 2: Pitching (N parallel subagents, where N = number of gremlins in config)
One subagent per gremlin. Each receives:
- Context brief from Phase 1
- Their personality description from config
- Gremlin prompt template (pitch section)

Each picks ONE idea (feature or content) and writes a pitch to `runs/YYYY-MM-DD/pitches/{slug}.md`. If PR mode, orchestrator commits pitches after this phase.

### Phase 3: Cross-Critique (N parallel subagents)
Each gremlin reads ALL pitches. Each writes critiques of the OTHER gremlins' ideas to `runs/YYYY-MM-DD/critiques/{slug}.md`. Critiques stay in character:
- Chaos Agent pushes for more weirdness
- Perfectionist finds structural flaws and fixes
- End User asks "would I actually use this?"

If PR mode, orchestrator commits critiques after this phase.

### Phase 3.5: Deep Dive (Orchestrator)
Before Phase 4 fans out, the **orchestrator** does targeted exploration for each pitch — reading the specific files, types, and interfaces the idea would touch. Each deep dive is capped at ~400 words and contains code snippets, file paths, and extension points. For content pitches, the deep dive also includes the schema/type definition, 1-2 representative examples, and the registration point.

This phase exists for reliability: long subagent exploration followed by long subagent writing is the single most common failure mode (stream-idle timeouts). Moving the exploration to the orchestrator and passing a compact, pre-digested deep dive into Phase 4 dramatically reduces the failure rate.

### Phase 4: Design (N parallel subagents)
Each gremlin receives:
- Their original pitch
- Critiques the other gremlins wrote about it
- The deep dive the orchestrator prepared for this idea

The subagent writes a full design doc at `runs/YYYY-MM-DD/designs/{slug}-{idea-slug}.md` using a **skeleton-first-then-Edit-per-section** pattern: a single Write call creates the file with section headers only, then one Edit call per section fills the content. Short tool calls reset the idle clock and checkpoint progress to disk, so a mid-run timeout still leaves completed sections saved.

Length budgets: ≤ 1000 words for feature ideas, ≤ 1500 words for content ideas.

**Feature design sections:** Problem statement, Proposed solution, User experience, Approaches considered (2-3 with trade-offs), Impact assessment, Open questions for humans, Wildness rating (1-5), Why this is a gremlin idea.

**Content design sections:** Problem statement, Proposed solution, User experience, **Draft of the content** (in the shape of the schema), **Where it goes** (file path + registration steps), **Three example moments**, Open questions for humans, Wildness rating, Why this is a gremlin idea.

If a subagent times out or produces an empty file, the orchestrator respawns it once with a tightened prompt: critiques dropped, lower word cap (800/1200), single example moment for content ideas, but the skeleton-first-then-Edit pattern preserved. The orchestrator **never** writes long-form content inline in its own turn — that path causes the same stream-idle timeouts. If PR mode, orchestrator commits designs after this phase.

### Phase 5: Summary & Delivery (Orchestrator)
- Reads all design docs
- Writes `runs/YYYY-MM-DD/README.md` with per-idea summaries and recommendation
- Updates `seed-ideas.md` (moves used seeds to "Previously Used")
- Delivers based on `output.mode`:
  - **local:** Done — prints summary to terminal
  - **pr:** Commits the README, pushes the branch (created in Setup), opens a PR
  - **issue:** Posts summary as GitHub issue

### Subagent tool access
- **Phase 2 (pitch):** Read, Glob, Grep, Write
- **Phase 3 (critique):** Write only
- **Phase 4 (design):** Read, Write, Edit (Read and Edit are required for the skeleton-first-then-Edit-per-section pattern)
- No destructive tools. Agents never modify existing project files.

---

## Prompt Files

### `orchestrator-prompt.md`

The orchestrator agent follows this file. Contains:

1. **Setup instructions** — read config, create output directories
2. **Phase 1 survey instructions** — what to explore, how to build the context brief, referencing `explore.paths` and `explore.context` from config
3. **Phase 2-4 subagent dispatch instructions** — for each phase, spawn N parallel agents using the gremlin prompt template with these placeholders filled in:
   - `{{PERSONALITY_NAME}}`, `{{PERSONALITY_DESCRIPTION}}` — from config
   - `{{CONTEXT_BRIEF}}` — built during Phase 1
   - `{{OUTPUT_PATH}}` — where to write results
   - `{{ALL_PITCHES}}` — (Phase 3) content of all pitch files
   - `{{MY_PITCH}}` — (Phase 4) the gremlin's own pitch
   - `{{CRITIQUES_OF_MY_PITCH}}` — (Phase 4) critiques from other gremlins
   - `{{DEEP_DIVE}}` — (Phase 4) the targeted deep dive the orchestrator produced in Phase 3.5 for this idea
4. **Phase 3.5 deep-dive instructions** — orchestrator-side targeted exploration of files/types relevant to each pitch, capped at ~400 words per idea
5. **Phase 5 summary instructions** — compile README, deliver per output mode, update seed tracking

### `gremlin-prompt-template.md`

Single template used for all three subagent phases. The orchestrator selects the appropriate phase section when constructing each subagent's prompt:

- **Identity section** — "You are {{PERSONALITY_NAME}}. {{PERSONALITY_DESCRIPTION}}" + framing on features-vs-content
- **Context section** — the context brief
- **Pitch instructions** (Phase 2) — explore through your personality lens, pick one idea (feature or content), write a pitch
- **Critique instructions** (Phase 3) — read all pitches, critique the others in character
- **Design instructions** (Phase 4) — read critiques of your pitch + the orchestrator's deep dive, produce a full design doc using skeleton-first-then-Edit-per-section. Sections differ for feature vs content ideas.

Placeholder substitution is simple string replacement — no templating engine.

---

## Scheduling (`gremlins schedule`)

Thin wrapper around Claude Code Remote Triggers.

Interactive setup:
1. **Frequency** — weekly (default), biweekly, monthly, or custom cron
2. **Timing** — day and time (converted to UTC)

Creates a Remote Trigger with prompt: "Read `gremlins/orchestrator-prompt.md` and follow its instructions exactly."

Also supports:
- `/gremlins schedule list` — shows current trigger
- `/gremlins schedule remove` — removes the trigger

---

## Prerequisites

- Claude Code with Agent tool support
- Opus recommended, Sonnet works for budget-conscious runs
- A project directory (git repo only required for PR output mode)
- GitHub CLI (`gh`) only if using PR or issue output mode
- No npm packages, no build step, no runtime dependencies

---

## What's NOT in Scope

- Custom phase structures (the 5-phase workflow is fixed)
- Non-Claude model support
- Web UI or dashboard
- Multi-repo exploration (one project at a time)
- Automatic implementation of generated ideas
