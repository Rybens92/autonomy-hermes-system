# consolidator_context.sh ⚠️ DISABLED — enable only with observer

## Purpose
Merges raw observation files into topic files. Runs every 12h. LLM job — the agent groups observations by theme and consolidates them.

**Disabled 2026-07-10.** Requires observer_context.sh to be active first (it consumes observation files). Enable both together or neither.

## Key Logic

### Context (lines 5-17)
Lists observation files filtered by age — only files older than **6 hours** are shown (`find -mmin +360`, line 8). This prevents recently-written observations from being consolidated before the agent has had time to process them in a subsequent tick.

Results are sorted by modification time and then **capped to the last 2 files** (`sort | tail -2`). Each file is injected with its filename as a heading, and content is **capped to the first 20 lines** (`head -20`).

**Optimization note (Task 8):** Previously all matching observation files were injected in full. Some files were 8-17KB (e.g., memory-analysis.md at 17.5KB). Capping to last 2 files + 20 lines each reduced output from 53 lines / 3861 bytes (variable, growing unbounded) to stable 40 lines / ~2500 bytes (34% reduction today, unlimited reduction on future growth).

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
