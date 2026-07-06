# cleanup.sh

## Purpose
Periodic maintenance for the autonomous system. Archives old done tasks, reports stuck blocked tasks, removes empty observation files and stale temp files. Runs as no-agent (zero LLM tokens). Produces output only when something was actually cleaned — empty stdout = silent (watchdog pattern).

## Key Logic

The script performs 5 sequential maintenance steps:

### 1. Kanban GC — 7-day retention (lines 12-19)
Runs `hermes kanban gc --event-retention-days 7 --log-retention-days 7` to purge events and worker logs older than 7 days. Only reports if the GC actually changed something (compares kanban stats before and after).

### 2. Archive done tasks older than 24h (lines 21-53)
Lists all done tasks via `kanban list --status done --json`, then uses an inline Python script to filter for tasks whose `completed_at` (or `updated_at`/`created_at` fallback) is older than 24 hours. Each matching task is archived via `kanban archive <id>`. The count of archived tasks is reported.

### 3. Report blocked tasks older than 12h (lines 55-82)
Lists all blocked tasks via `kanban list --status blocked --json`, then uses an inline Python script to count tasks whose `updated_at` (or `created_at` fallback) is older than 12 hours. If any are found, outputs a warning with the count: `⚠️ N task(s) blocked >12h — may need attention`. Does NOT delete or modify blocked tasks — only reports them.

### 4. Delete empty observation files (lines 84-90)
Finds and deletes empty `*.md` files in `observations/` using `find -empty -delete`. **No age filter** — any file that is empty gets removed regardless of age.

### 5. Delete stale /tmp files older than 24h (lines 92-98)
Finds and deletes `/tmp/hermes_*` files older than 24 hours (`find -mmin +1440 -delete`).

### Output Guard (lines 100-106)
Only outputs when `ACTION_TAKEN=1`. The report includes a timestamp and lines like:
```
✓ kanban gc (7d retention) — cleaned events/logs
✓ Archived 3 done task(s) older than 24h
⚠️ 1 task(s) blocked >12h — may need attention
✓ Removed 2 empty observation file(s)
✓ Cleaned 1 stale /tmp file(s)
Timestamp: 2026-07-06 12:00:00
```

Empty stdout = silent (no message sent to user).

## Modification Notes
- Adjust `--event-retention-days` and `--log-retention-days` for kanban GC to control how long history is kept
- Adjust the `timedelta(hours=24)` threshold for done-task archival
- Adjust the `timedelta(hours=12)` threshold for blocked-task reporting
- Empty observation cleanup uses no age filter — any empty `.md` file is deleted immediately. Add `-mmin +N` if you want a grace period
- Stale /tmp cleanup threshold (`-mmin +1440`) = 24 hours. Change to `+2880` for 48h
