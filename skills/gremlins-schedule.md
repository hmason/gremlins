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
