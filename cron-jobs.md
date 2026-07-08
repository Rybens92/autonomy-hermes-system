# Cron Jobs

All 8 cron jobs run under the `aa-orchestrator` profile. The gateway scheduler fires them automatically.

> **Note on profile naming**: The `aa-` prefix is the documented convention for fresh installs. On the creator's machine, these jobs actually run under the `autonomous-orchestrator` profile (no `aa-` prefix), because cron jobs are created via `hermes cron create` under whichever profile runs the command. Adjust the profile name in your setup to match your actual Hermes profile.

## Schedule Rationale

The schedule is designed around an **hourly orchestration tick** — the orchestrator evaluates the system state once per hour. Supporting jobs are offset to run between orchestration ticks.

```
XX:00  orchestrator-heartbeat         (LLM, ~700 tokens) — main decision loop
XX:05  memory-observer                (LLM, ~500 tokens) — capture patterns
XX:10  milestone-reporter             (no-agent, 0 tokens) — report changes
every 10 min: ping-human-watchdog     (no-agent, 0 tokens) — monitor for help requests
00:00, 12:00: memory-consolidator     (LLM, ~400 tokens) — merge observations
00:00, 06:00, 12:00, 18:00: cleanup   (no-agent, 0 tokens) — maintenance
18:00: daily-research                 (LLM, ~2000 tokens) — find interesting content
18:30: research-proposal-reporter     (no-agent, 0 tokens) — deliver proposals
```

## Job Details

| Job | Schedule | Type | Tokens/tick | Deliver | Script | Description |
|-----|----------|------|-------------|---------|--------|-------------|
| orchestrator-heartbeat | `0 * * * *` | LLM | ~700 | local | `state_context.sh` | Main decision loop — evaluates state, creates kanban cards |
| memory-observer | `5 * * * *` | LLM | ~500 | local | `observer_context.sh` | Captures system behavior patterns as observations |
| memory-consolidator | `0 */12 * * *` | LLM | ~400 | local | `consolidator_context.sh` | Merges raw observations into topic files |
| ping-human-watchdog | `*/10 * * * *` | no-agent | **0** | messaging | `check_help.sh` | Monitors HELP_NEEDED.md — alerts user |
| milestone-reporter | `10 * * * *` | no-agent | **0** | messaging | `milestone_reporter.sh` | Reports STATUS.md changes (phase transitions) |
| cleanup-agent | `0 */6 * * *` | no-agent | **0** | messaging | `cleanup.sh` | Archives done tasks, removes stale files |
| daily-research | `0 18 * * *` | LLM | ~2000 | local | `research_context.sh` | Searches web for content matching user interests, writes summaries + proposals |
| research-proposal-reporter | `30 18 * * *` | no-agent | **0** | messaging | `research_proposal_reporter.sh` | Delivers today's research proposals to user |

## Token Budget

With hourly orchestration ticks, a typical daily budget for a strong reasoning model comfortably supports:

```
Heartbeat:    24 ticks × ~700   = ~17,000 tokens/day
Observer:     24 ticks × ~500   = ~12,000 tokens/day
Consolidator:  2 ticks × ~400   =   ~800 tokens/day
Research:      1 tick  × ~2000  =  ~2,000 tokens/day
───────────────────────────────────────────────────
TOTAL (strong model):           = ~32,000 tokens/day
No-agent jobs: 0 tokens (scripts only)
Worker execution: varies (runs on cheapest model, not strong model)
```
> Creator's setup: GLM-5.2 (5h/day budget) for STRONGEST tier, DeepSeek v4 for CHEAPEST.
> Your costs depend on YOUR provider pricing — adjust tick frequency accordingly.

## Tuning

- **Higher urgency**: Reduce heartbeat to `*/30 * * * *` (every 30 min) — doubles cost
- **Lower cost**: Increase heartbeat to `0 */2 * * *` (every 2h) — halves cost
- **Less noise**: Increase watchdog to `*/30 * * * *` (every 30 min) — user gets help alerts less frequently
- **More research**: Add a second daily research tick or make it more frequent
