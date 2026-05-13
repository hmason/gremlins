---
description: Run gremlins — autonomous idea-generating agents explore your project and produce design documents
---

# Gremlins Run

## Pre-flight Check

1. Check if `gremlins/gremlins.yaml` exists. If not, tell the user: "Gremlins isn't set up yet. Run `/gremlins:init` first." and stop.
2. Check if `gremlins/orchestrator-prompt.md` exists. If not, tell the user: "Missing orchestrator prompt at `gremlins/orchestrator-prompt.md`. Run `/gremlins:init` to regenerate." and stop.
3. Check if `gremlins/gremlin-prompt-template.md` exists. If not, tell the user: "Missing gremlin prompt template at `gremlins/gremlin-prompt-template.md`. Run `/gremlins:init` to regenerate." and stop.

## Execute

Read `gremlins/orchestrator-prompt.md` and follow its instructions exactly.

The orchestrator prompt contains the full five-phase workflow:
1. Survey the project (read-only exploration)
2. Spawn parallel subagents to pitch ideas
3. Spawn parallel subagents for cross-critique
4. Spawn parallel subagents to produce full design documents
5. Compile summary and deliver results

The orchestrator prompt references `gremlins/gremlins.yaml` for configuration and `gremlins/gremlin-prompt-template.md` for subagent prompts. Follow whatever is written in those files — the user may have customized them.
