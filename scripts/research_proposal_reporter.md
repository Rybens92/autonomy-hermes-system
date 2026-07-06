# research_proposal_reporter.sh

## Purpose
Delivers today's research proposals to the user's messaging platform. Runs as no-agent (zero LLM tokens). 30 minutes after the research agent finishes (18:30).

## Key Logic

### Detection (line 7)
Checks if today's proposals file exists and is non-empty. If not → silent exit.

### Output (line 10-12)
Prints header + full proposal content. Stdout delivered verbatim to user.

## Modification Notes
- Change the schedule if you want proposals delivered at a different time
- Add filtering: only deliver proposals with `Impact: high` or above a certain threshold
- If proposals accumulate multiple days, add a "recent proposals" summary mode
