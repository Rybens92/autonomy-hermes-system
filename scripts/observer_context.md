# observer_context.sh

## Purpose
Captures noteworthy system events into observation files for later consolidation. LLM job — the agent analyzes recent behavior and writes structured observations.

## Key Logic

### Context (lines 5-13)
Injects **only** the last 50 lines of LOG.md (`tail -50`, line 8) as context. No STATUS.md is injected.

Also injects the most recent observation file under `[LAST OBSERVATION]` (lines 10-11) — reads the newest `*.md` file in `observations/`. If no observations exist yet, shows `(none yet)`.

### Task (lines 15-20)
Instructs the agent to:
1. Read the LOG above and extract **atomic** observations from **new entries** (not patterns — individual facts)
2. Write ONE observation per fact — each: timestamp + **ONE factual sentence** (not prose, not summary, not 2-5 sentences)
3. Save to `observations/YYYY-MM-DD_HH-MM.md` (format: `%Y-%m-%d_%H-%M`)
4. Only facts — no interpretation, no filler

### Observation filename format
- Live script uses `date +%Y-%m-%d_%H-%M` — e.g., `2026-07-06_14-30.md`
- Between hour and minute: **hyphen** (`-`), not concatenated digits

## Modification Notes
- Adjust LOG.md tail length (`tail -50`) based on tick frequency. Every-30min ticks with 1 decision each = ~48 lines/day
- If the observer produces too many observations, add a "max 3 observations per run" limit
- The observation format can be structured (YAML frontmatter) if you want programmatic analysis later
- The `[LAST OBSERVATION]` injection helps the agent avoid re-observing the same events
