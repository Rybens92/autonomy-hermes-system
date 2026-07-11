# Cron Jobs

6 active + 2 disabled cron jobs run under the `aa-orchestrator` profile. The gateway scheduler fires them automatically. **The `orchestrator-heartbeat` uses skills: ["kanban-pipeline-orchestration"]** (see jobs.json).

**Stable architecture (P0 2026-07-10 verification):** 
- aa-orchestrator dual role (cron heartbeat + Telegram chat/user comms)
- Perpetual continuous audit when idle
- v2 pipeline (explorer + researcher + planner + plan-taste + worker + ...)
- Goal 4 (aa-comms / headless split) abandoned — not implemented. No separate comms profile.

> **Note on profile naming**: The `aa-` prefix is the documented convention for fresh installs. On the creator's machine, these jobs actually run under the `autonomous-orchestrator` profile (no `aa-` prefix), because cron jobs are created via `hermes cron create` under whichever profile runs the command. Adjust the profile name in your setup to match your actual Hermes profile.

## Schedule Rationale

The schedule is designed around an **hourly orchestration tick** — the orchestrator evaluates the system state once per hour. Supporting jobs are offset to run between orchestration ticks.

```
XX:00  orchestrator-heartbeat         (LLM, ~700 tokens) — main decision loop
XX:05  memory-observer                (LLM, ~500 tokens) — DISABLED
XX:10  milestone-reporter             (no-agent, 0 tokens) — report changes
every 10 min: ping-human-watchdog     (no-agent, 0 tokens) — monitor for help requests
00:00, 12:00: memory-consolidator     (LLM, ~400 tokens) — DISABLED
00:00, 06:00, 12:00, 18:00: cleanup   (no-agent, 0 tokens) — maintenance
18:00: daily-research                 (LLM, ~2000 tokens) — find interesting content
18:30: research-proposal-reporter     (no-agent, 0 tokens) — deliver proposals
```

## Job Details

### Active (6)
| Job | Schedule | Type |
|-----|----------|------|
| orchestrator-heartbeat | 0 * * * * | LLM |
| ping-human-watchdog | */10 * * * * | no-agent |
| milestone-reporter | 10 * * * * | no-agent |
| cleanup-agent | 0 */6 * * * | no-agent |
| daily-research | 0 18 * * * | LLM |
| research-proposal-reporter | 30 18 * * * | no-agent |

### Disabled (2)
| Job | Reason |
|-----|--------|
| memory-observer | Produces observations nobody consumes — enable only if memory system is wired |
| memory-consolidator | Merges observations nobody reads — enable only with observer |

| Job | Schedule | Type | Tokens/tick | Deliver | Script | Description |
|-----|----------|------|-------------|---------|--------|-------------|
| orchestrator-heartbeat | `0 * * * *` | LLM | ~700 | local | `state_context.sh` | Main decision loop — evaluates state, creates kanban cards |
| ⚠️ DISABLED memory-observer | `5 * * * *` | LLM | ~500 | local | `observer_context.sh` | Captures system behavior patterns as observations |
| ⚠️ DISABLED memory-consolidator | `0 */12 * * *` | LLM | ~400 | local | `consolidator_context.sh` | Merges raw observations into topic files |
| ping-human-watchdog | `*/10 * * * *` | no-agent | **0** | messaging | `check_help.sh` | Monitors HELP_NEEDED.md — alerts user |
| milestone-reporter | `10 * * * *` | no-agent | **0** | messaging | `milestone_reporter.sh` | Reports STATUS.md changes (phase transitions) |
| cleanup-agent | `0 */6 * * *` | no-agent | **0** | local | `cleanup.sh` | Archives done tasks, removes stale files |
| daily-research | `0 18 * * *` | LLM | ~2000 | local | `research_context.sh` | Searches web for content matching user interests, writes summaries + proposals |
| research-proposal-reporter | `30 18 * * *` | no-agent | **0** | messaging | `research_proposal_reporter.sh` | Delivers today's research proposals to user |

## Token Budget

With hourly orchestration ticks, a typical daily budget for a strong reasoning model comfortably supports:

```
Heartbeat:    24 ticks × ~700   = ~17,000 tokens/day
Research:      1 tick  × ~2000  =  ~2,000 tokens/day
───────────────────────────────────────────────────
ACTIVE TOTAL (strong model):      = ~19,000 tokens/day
No-agent jobs: 0 tokens (scripts only)
Worker execution: varies (runs on cheapest model, not strong model)
```

### Disabled (would add if re-enabled)
```
Observer:     24 ticks × ~500   = ~12,000 tokens/day
Consolidator:  2 ticks × ~400   =   ~800 tokens/day
───────────────────────────────────────────────────
DISABLED TOTAL:                   = ~12,800 tokens/day
```
If you re-enable observer + consolidator, add ~12,800/day.

> Project creator uses STRONGEST reasoning model / CHEAPEST execution model. Your costs depend on YOUR provider pricing — adjust tick frequency accordingly.

## Tuning

- **Higher urgency**: Reduce heartbeat to `*/30 * * * *` (every 30 min) — doubles cost
- **Lower cost**: Increase heartbeat to `0 */2 * * *` (every 2h) — halves cost
- **Less noise**: Increase watchdog to `*/30 * * * *` (every 30 min) — user gets help alerts less frequently
- **More research**: Add a second daily research tick or make it more frequent
