# Gremlins

Autonomous bad/brilliant idea generation agents for Claude Code. Gremlins explores your project with distinct AI personalities, then produces creative design documents through a structured pitch → critique → design workflow.

This project was inspired by [Nightshift](https://github.com/marcus/nightshift), except instead of using your extra token budget for banal code maintentance tasks, it uses it to explore the edge of the idea space for your product.

## How it works

1. **Survey** — Gremlins reads your project to understand what exists and what matters
2. **Pitch** — Each gremlin independently picks and pitches one wild idea
3. **Critique** — Gremlins cross-critique each other's pitches, staying in character
4. **Design** — Each gremlin produces a full design document incorporating feedback
5. **Deliver** — A summary with all designs is delivered as local files, a PR, or a GitHub issue

Ideas can be **features** (new capabilities) or **content** (a specific new instance of an existing data type — a new template, a new entry in a registry, a new value in an enum). Both are first-class outputs. If your project has a content surface, gremlins will pitch into it.

## Installation

Add the marketplace, then install:

```bash
claude plugin marketplace add https://github.com/hmason/gremlins
claude plugin install gremlins
```

Or load it for a single session without installing (handy for trying it out or editing the prompts):

```bash
git clone https://github.com/hmason/gremlins
claude --plugin-dir ./gremlins
```

## Quick Start

```
# Set up gremlins in your project (interactive wizard)
/gremlins:init

# Run gremlins now
/gremlins:run

# Schedule recurring runs
/gremlins:schedule
```

## Default Personalities

- **The Chaos Agent** — lives for the weird and unexpected. Motto: "What's the worst that could happen?"
- **The Perfectionist** — finds structural flaws AND elegant fixes. Motto: "Yes, but have you considered..."
- **The End User** — only cares about the lived experience. Motto: "But would I actually use this?"

You can customize personalities, add new ones, or start from scratch during setup.

## Configuration

After running `/gremlins:init`, your project gets a `gremlins/` directory:

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
