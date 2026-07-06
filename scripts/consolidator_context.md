# consolidator_context.sh

## Purpose
Merges raw observation files into topic files. Runs every 12h. LLM job — the agent groups observations by theme and consolidates them.

## Key Logic

### Context (lines 5-17)
Lists observation files filtered by age — only files older than **6 hours** are shown (`find -mmin +360`, line 8). This prevents recently-written observations from being consolidated before the agent has had time to process them in a subsequent tick.

Each matching file is injected in full with its filename as a heading (lines 8-12).

Also lists existing topic files in `observations/topics/` (line 15). If none exist, shows `(none)`.

### Task (lines 19-24)
Instructs the agent to:
1. Group observations by theme
2. Create or append to `observations/topics/THEME.md`
3. Delete original observation files after consolidation
4. Compress without information loss

### Age filter detail
- `find -mmin +360` = files whose last modification time is more than 360 minutes (6 hours) ago
- Only observations passing this filter are injected as context
- This means the consolidator only processes observations that are "settled" — at least 6 ticks old

## Modification Notes
- If observation files are large, add a size limit to prevent context overflow
- The "delete after consolidation" step is critical — without it, observations accumulate and get re-consolidated
- Add a backup step before deletion if you want an audit trail
- Adjust the `-mmin +360` threshold to change how long observations sit before consolidation (e.g., `-mmin +720` for 12h)
