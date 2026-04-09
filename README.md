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
