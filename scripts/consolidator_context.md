# consolidator_context.sh

## Purpose
Merges raw observation files into topic files. Runs every 12h. LLM job — the agent groups observations by theme and consolidates them.

## Key Logic

### Context (line 6-18)
Lists all raw observation files (`.md` in `observations/`) and existing topic files (`.md` in `observations/topics/`). Both are injected in full.

### Task (line 20-25)
Instructs the agent to:
1. Group observations by theme
2. Create or append to `observations/topics/THEME.md`
3. Delete original observation files after consolidation
4. Compress without information loss

## Modification Notes
- If observation files are large, add a size limit to prevent context overflow
- The "delete after consolidation" step is critical — without it, observations accumulate and get re-consolidated
- Add a backup step before deletion if you want an audit trail
