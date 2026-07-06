# observer_context.sh

## Purpose
Captures noteworthy system events into observation files for later consolidation. LLM job — the agent analyzes recent behavior and writes structured observations.

## Key Logic

### Context (line 6-10)
Injects STATUS.md and last 30 lines of LOG.md as context.

### Task (line 12-17)
Instructs the agent to:
1. Identify patterns (successes, failures, decisions)
2. Write a concise observation per pattern to `observations/YYYY-MM-DD_HHMM.md`
3. Each observation: date, one-line summary, 2-5 sentences of context
4. Skip if nothing noteworthy

## Modification Notes
- Adjust LOG.md tail length (`tail -30`) based on tick frequency. Hourly ticks with 1 decision each = ~24 lines/day
- If the observer produces too many observations, add a "max 3 observations per run" limit
- The observation format can be structured (YAML frontmatter) if you want programmatic analysis later
