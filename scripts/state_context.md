# state_context.sh

## Purpose
Injected as context before every orchestrator tick. Reads all state files, kanban board, and cron status. The orchestrator sees this as its initial prompt input.

## Key Logic

### HELP_NEEDED Guard (line 9-13)
If HELP_NEEDED.md is non-empty, the script outputs a TINY prompt (~200 bytes):
```
Blocker detected: HELP_NEEDED.md is non-empty.
Do nothing this tick. No tool calls. No file writes. Exit immediately.
```
This saves ~95% tokens compared to full context injection when the system is blocked.

### Normal Context (line 16-44)
When not blocked, injects:
- GOAL.md — user's goals
- STATUS.md — current system state
- LOG.md — last 20 decision lines
- PROJECT.md — workspace path
- HELP_NEEDED.md — escalation status
- Kanban board — task list
- Cron status — job health

### Task Instruction (line 45-49)
Appends a concise task instruction telling the orchestrator what to evaluate and how to create cards.

## Modification Notes
- The PROJECT.md path is used as `workspace_path` in kanban_create — ensure PROJECT.md contains the correct absolute path
- The `HERMES_BIN` variable must point to the hermes CLI binary. Adjust based on installation.
- HELP_NEEDED guard can be adjusted: change the minimal prompt text, or add a cooldown (e.g., only inject full context every Nth tick when blocked)
