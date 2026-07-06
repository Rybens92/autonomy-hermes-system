# cleanup.sh

## Purpose
Periodic maintenance for the autonomous system. Archives old done tasks, alerts on stuck blocked tasks, removes empty observation files and stale temp files. Runs as no-agent (zero LLM tokens).

## Key Logic

### Kanban GC (line 9)
Runs `hermes kanban gc --older-than 24h` to archive completed tasks. The kanban system handles the actual archival; this script just triggers it.

### Blocked Task Alert (line 11-14)
Checks for blocked tasks. If any exist, outputs a warning. No filtering by age — all blocked tasks are reported.

### File Cleanup (line 16-18)
- Removes empty observation files older than 24h
- Removes stale `/tmp/hermes_*` temp files older than 24h

### Output Guard (line 20-21)
Only outputs when something was changed. Empty stdout = silent (no message to user).

## Modification Notes
- Adjust `--older-than` threshold for kanban gc based on how long you want to keep done tasks visible
- Add blocked task age filtering: `kanban list | grep blocked | grep ">12h"` to only alert on long-stuck tasks
- Add disk space check if workspace is on a limited partition
