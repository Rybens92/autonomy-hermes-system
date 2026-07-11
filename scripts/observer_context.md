# observer_context.sh ⚠️ DISABLED — enable only if memory system is wired

## Purpose
Captures noteworthy system events into observation files for later consolidation. LLM job — the agent analyzes recent behavior and writes structured observations.

**Disabled 2026-07-10** — the observations it produces have no downstream consumer (memory-consolidator is also disabled). Enable both together or neither.

## Key Logic

### Context (lines 5-13)
Injects **only** the last 10 decision log entries (`grep "^\[" LOG.md | tail -10`, line 8) as context. LOG.md entries are filtered to timestamped lines only. No STATUS.md is injected.

**Optimization note (Task 8):** Previously injected `tail -50` (~31KB from long decision lines). Filtering to `^[` + capping at 10 lines reduced output from 128 lines / 38649 bytes to 54 lines / 11584 bytes (70% reduction).

Also injects the most recent observation file under `[LAST OBSERVATION]` (lines 10-11) — reads the newest `*.md` file in `observations/`. Capped to first 30 lines via `head -30` to prevent large observation files from flooding context. If no observations exist yet, shows `(none yet)`.

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
