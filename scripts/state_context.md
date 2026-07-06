# state_context.sh

## Purpose
Injected as context before every orchestrator tick. Reads all state files, kanban board, and help-needed status. The orchestrator sees this as its initial prompt input.

## Key Logic

### HELP_NEEDED Guard (lines 11-24)
If HELP_NEEDED.md is non-empty, the script outputs a detailed guard block (~600 bytes) and exits immediately — **no full context is injected**:

```
=== HELP_NEEDED: WAITING FOR USER ===

[BLOCKER CONTENT]
<contents of HELP_NEEDED.md>

=== YOUR TASK ===
You are STILL waiting for the user to respond to the blocker above.
DO NOT create any kanban cards. DO NOT update STATUS.md or LOG.md.
DO NOT call any write tools. Just acknowledge you're waiting.
If the blocker content has changed (user responded), evaluate it.
OTHERWISE: respond with 'IDLE — waiting for user.' and no tool calls.
```

This saves ~95% tokens compared to full context injection when the system is blocked.

### Normal Context (lines 26-46)
When not blocked, injects:
- PROJECT.md — workspace path (line 29)
- GOAL.md — user's goals (line 32)
- STATUS.md — current system state (line 35)
- LOG.md — last **30** lines of decision log (`tail -30`, line 38)
- HELP_NEEDED.md — escalation status (shows `(none)` when absent, line 41)
- Kanban board — task list via `kanban list | head -30` (line 44)

No cron status is output — the script does not query or report cron job health.

### Task Instruction (lines 48-55)
Appends a concise task instruction telling the orchestrator to:
1. Create one kanban card if there is work to do (use `workspace_path` from PROJECT.md)
2. Update STATUS.md with current state
3. Append a line to LOG.md: `[TIMESTAMP] [DECISION] description`
4. If nothing to do: update STATUS.md as idle and finish
5. If completely blocked: write to HELP_NEEDED.md

## Modification Notes
- The PROJECT.md path is used as `workspace_path` in kanban_create — ensure PROJECT.md contains the correct absolute path
- The `HERMES_BIN` variable must point to the hermes CLI binary. Adjust based on installation.
- HELP_NEEDED guard can be adjusted: change the guard text, or add a cooldown (e.g., only inject full context every Nth tick when blocked)
- Adjust `tail -30` LOG.md line count for higher or lower tick frequency
