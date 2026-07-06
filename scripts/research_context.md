# research_context.sh

## Purpose
Daily research agent — searches the web for recent content matching user interests, produces summaries and actionable proposals. LLM job (once daily at 18:00).

## Key Logic

### Context (line 7-11)
Injects:
- `research/interests.md` — user's interest topics
- `research/topic-index.md` — topics already covered (dedup)

### Task (line 13-22)
Instructs the agent to:
1. Search the web for each interest topic
2. Skip topics already in topic-index.md
3. Write 3-5 sentence summaries to `research/daily/YYYY-MM-DD.md`
4. Create proposals to `research/proposals/YYYY-MM-DD.md` with impact/effort estimates
5. Update topic-index.md with new topics covered

## Modification Notes
- The agent needs `web` toolset to search — ensure the orchestrator profile's toolsets include it (it does, through the cron job inheriting the profile config)
- If the search produces too many results, add a limit: "max 3 findings per interest topic"
- The proposals format can be extended with custom fields (dependencies, prerequisites, etc.)
