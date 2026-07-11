# research_context.sh

## Purpose
Daily research agent — searches the web for recent content matching user interests, produces summaries and actionable proposals. LLM job (once daily at 18:00).

## Key Logic

### Context (lines 8-27)
Injects:
- `research/interests.md` — user's interest topics (short, ~16 lines — kept in full)
- `research/topic-index.md` — only the **last 5 topics** (`grep -A 6 "^## " | head -35`), trimmed from the full 143-line file to ~35 lines
- Last 3 daily reports — first **10 lines** each (`head -10` per file), reduced from 20 lines to save ~100 tokens per report

**Optimization note (Task 8):** Topic-index trimmed to last 5 entries (was all 143 lines). Report headers capped to 10 lines each. Total savings: 211→93 lines, 9370→4453 bytes (52% reduction).

### Task (lines 29-50)
Instructs the agent to:
1. Search the web for each interest topic
2. Skip topics already in topic-index.md (cross-reference the injected last 5 entries)
3. Search for 3 content types: 🔥 TRENDS, 🆕 NEWS, 🔍 NICHES
4. Save full report to `research/daily/YYYY-MM-DD.md`
5. Update topic-index.md with new topics covered
6. Create proposals to `research/proposals/YYYY-MM-DD.md` with project ideas and GOAL.md additions

## Modification Notes
- The agent needs `web` toolset to search — ensure the research agent profile's toolsets include it (it does, through the cron job inheriting the profile config)
- If the search produces too many results, add a limit: "max 3 findings per interest topic"
- The proposals format can be extended with custom fields (dependencies, prerequisites, etc.)
