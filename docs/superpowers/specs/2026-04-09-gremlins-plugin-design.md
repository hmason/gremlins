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

## 5-Phase Orchestration (`gremlins run`)

### Phase 1: Survey (Orchestrator, read-only)
- Reads `gremlins.yaml` for config
- Reads `seed-ideas.md` if present
- Explores paths in `explore.paths`, skipping `explore.ignore`
- Reads `explore.context` to understand what matters
- Checks previous runs in `output.dir` to avoid repeating ideas
- Produces an in-memory context brief

### Phase 2: Pitching (N parallel subagents, where N = number of gremlins in config)
One subagent per gremlin. Each receives:
- Context brief from Phase 1
- Their personality description from config
- Gremlin prompt template (pitch section)

Each picks ONE idea and writes a pitch to `runs/YYYY-MM-DD/pitches/{slug}.md`.

### Phase 3: Cross-Critique (N parallel subagents)
Each gremlin reads ALL pitches. Each writes critiques of the OTHER gremlins' ideas to `runs/YYYY-MM-DD/critiques/{slug}.md`. Critiques stay in character:
- Chaos Agent pushes for more weirdness
- Perfectionist finds structural flaws and fixes
- End User asks "would I actually use this?"

### Phase 4: Design (N parallel subagents)
Each gremlin reads critiques others wrote about THEIR pitch. Produces a full design doc at `runs/YYYY-MM-DD/designs/{slug}-{idea-slug}.md`:

- Problem/opportunity statement
- Proposed solution
- User experience description
- 2-3 approaches considered with trade-offs
- Recommended approach with reasoning
- Impact assessment (effort, risk, dependencies)
- Open questions for humans
- "Why this is a gremlin idea" — what assumption it challenges

### Phase 5: Summary & Delivery (Orchestrator)
- Reads all design docs
- Writes `runs/YYYY-MM-DD/README.md` with per-idea summaries and recommendation
- Updates `seed-ideas.md` (moves used seeds to "Previously Used")
- Delivers based on `output.mode`:
  - **local:** Done — prints summary to terminal
  - **pr:** Creates branch `gremlins/YYYY-MM-DD`, commits, opens PR
  - **issue:** Posts summary as GitHub issue

### Subagent tool access
- Read, Glob, Grep — codebase exploration
- Write — output files only
- No destructive tools. Agents never modify existing code.

---

## Prompt Files

### `orchestrator-prompt.md`

The orchestrator agent follows this file. Contains:

1. **Setup instructions** — read config, create output directories
2. **Phase 1 survey instructions** — what to explore, how to build the context brief, referencing `explore.paths` and `explore.context` from config
3. **Phase 2-4 subagent dispatch instructions** — for each phase, spawn N parallel agents using the gremlin prompt template with these placeholders filled in:
   - `{{PERSONALITY_NAME}}`, `{{PERSONALITY_DESCRIPTION}}` — from config
   - `{{CONTEXT_BRIEF}}` — built during Phase 1
   - `{{PHASE}}` — "pitch", "critique", or "design"
   - `{{OUTPUT_PATH}}` — where to write results
   - `{{ALL_PITCHES}}` — (Phase 3+) content of all pitch files
   - `{{CRITIQUES_OF_MY_PITCH}}` — (Phase 4) critiques from other gremlins
4. **Phase 5 summary instructions** — compile README, deliver per output mode, update seed tracking

### `gremlin-prompt-template.md`

Single template used for all three subagent phases, with sections gated by `{{PHASE}}`:

- **Identity section** — "You are {{PERSONALITY_NAME}}. {{PERSONALITY_DESCRIPTION}}"
- **Context section** — the context brief
- **Pitch instructions** (Phase 2) — explore through your personality lens, pick one idea, write a pitch
- **Critique instructions** (Phase 3) — read all pitches, critique the others in character
- **Design instructions** (Phase 4) — read critiques of your pitch, deep dive, produce full design doc

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
