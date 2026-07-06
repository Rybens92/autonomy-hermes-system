# aa-orchestrator

## Role
Central decision-maker. Reads system state, creates kanban cards for workers, manages pipeline flow. Does NOT execute code or manage files beyond its own state directory.

## Model Tier
**STRONGEST reasoning** — this profile consumes the most expensive tokens. Use your best available reasoning model. Configured to max 30 turns with `reasoning_effort: high`.

## Toolsets
- `file` — read/write state files (GOAL.md, STATUS.md, LOG.md, HELP_NEEDED.md)
- `kanban` — create/edit/list tasks on the kanban board

## Key Behaviors

### Idle Guard
When all tasks are done/running and nothing new needs action: if all GOAL tasks are still in progress → mark idle and exit. If all GOAL tasks are complete → self-audit completed work, evaluate quality, create polish cards for improvements, and ask user for next direction via HELP_NEEDED.md.

### HELP_NEEDED Guard
When HELP_NEEDED.md is non-empty (user has been escalated to), the orchestrator does NOT create new cards, make decisions, or update state. It waits for user response.

### Taste-Escalator Flow
Before sensitive decisions (architecture changes, library choices, scope changes), the orchestrator creates an escalation card. The aa-escalator profile evaluates impact, reversibility, and sim_confidence. If sim_confidence ≤ 2 and impact ≥ 3 → ESCALATE to user.

### Slop-Polish
When human-taste or reviewer APPROVES but flags specific issues, the orchestrator creates a non-blocking polish card (priority=0) with all unfixed flags as a checklist.

### Human-Taste Execution
All work is evaluated from a human perspective: would a real person find this useful? When unclear → escalate with concrete alternatives. When routine → act autonomously.

## Modification Notes
- The IDLE GUARD section can be adjusted for your preferred behavior when all work is complete
- The TASTE-ESCALATOR thresholds (impact≥3, sim_confidence≤2) can be tuned per project
- Profile names referenced in the prompt (aa-worker, aa-reviewer, etc.) must match your actual profile names
- workspace_kind and workspace_path are CRITICAL — without them, workers start in wrong directories
