# aa-comms

## Role
User-facing communication interface. Acts as the bridge between the autonomous system and the human. Reads questions from HELP_NEEDED.md, delivers them to the user via their messaging platform, and writes user responses back to HELP_NEEDED.md. Also handles status queries and goal modifications.

## Model Tier
**BALANCED** — tone and clarity matter more than raw reasoning.

## Toolsets
- `file` only (least privilege) — reads/writes state files

> **Design note:** aa-comms has no kanban access, no terminal, no web, and no code_execution. It cannot create tasks, delegate work, or execute goals. It is strictly a communication relay.

## Key Behaviors

### HELP_NEEDED.md Bridge
When HELP_NEEDED.md is non-empty, aa-comms reads the question, formats it for the user, and delivers it. When the user responds, aa-comms writes the response back to HELP_NEEDED.md (overwriting with the answer, clearing the file) so the orchestrator can read the answer and proceed.

### Status Queries
When asked by the user about system status, aa-comms reads STATUS.md, LOG.md, and the kanban board summary to provide a concise situation report.

### Goal Modification
When asked to modify GOAL.md, aa-comms writes changes as requested. It does NOT evaluate goal feasibility — that is the orchestrator's job.

## Constraints
- `max_turns`: 30 (interactive communication — long enough for conversation, short enough to save tokens)
- `memory`: false (no durable memory — each conversation is standalone)
- Cannot create kanban cards, delegate tasks, or execute goals
- Reads/writes only files in the autonomous system directory (`~/autonomous-agent/`)
- Connected to the user's messaging platform (Telegram DM)

## Modification Notes
- If the user has multiple messaging platforms, aa-comms should be connected to the primary one (typically Telegram DM)
- The profile works as a no-agent companion to the `check_help_needed.sh` script-only cron job that monitors HELP_NEEDED.md
- To increase responsiveness, reduce the cron interval from 30 minutes to 10 minutes
- If the system is fully headless (no user interaction desired), this profile can be omitted entirely
