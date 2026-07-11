# Autonomy Hermes System

Turn your [Hermes Agent](https://github.com/nousresearch/hermes-agent) into a fully autonomous agent that works on multi-phase projects, reports to you via your messaging platform, simulates human taste in decision-making, and requests your direction when it's stuck — all without modifying a single line of Hermes source code.

## What This Is

A drop-in configuration layer that sits on top of your existing Hermes installation. It uses **only built-in Hermes features**:

- **Cron jobs** — scheduled autonomous ticks (hourly by default)
- **Kanban board** — task dispatch across specialized worker profiles
- **Profiles** — different models/toolsets for different roles
- **Scripts** — context injection, watchdog, cleanup, reporting
- **Gateway** — messaging platform delivery (Telegram, Discord, Slack…)

## Architecture at a Glance (v2 + dual role)

```
GOAL.md              user writes WHAT to do
   │
   ▼
cron: heartbeat (aa-orchestrator, skill=kanban-pipeline-orchestration) ──── state_context.sh
   │
   ▼
aa-orchestrator (DUAL ROLE: cron orchestration + Telegram chat)
   │  reads state (GOAL/STATUS/LOG/HELP_NEEDED), manages kanban, perpetual audit
   │
   │    ┌──────────────────────────────────────┘
   │    ▼
   │  kanban dispatcher ─► aa-explorer (fs discovery, CHEAPEST)
   │                       aa-researcher (web+prior art, STRONGEST)
   │                       aa-planner (decompose, STRONGEST)
   │                       aa-plan-taste (plan verify, STRONGEST)
   │                       aa-worker (execute, CHEAPEST)
   │                       aa-reviewer (tech review)
   │                       aa-human-taste (slop/human judgment)
   │                       aa-verifier (facts)
   │                       aa-goalkeeper (final goal check)
   │                       aa-thinker (meta)
   │
   ▼
Gate decisions: run pipeline ──► approve ──► next phase
                  │
                  ▼
            blocker ──► HELP_NEEDED.md ──► watchdog ──► Telegram 🚨
```

**Stable architecture (P0 2026-07-10, 11 profiles):** aa-orchestrator dual role (Goal 4 / gateway-split abandoned). Perpetual continuous audit when idle. v2 pipeline deployed.

## Features

### Core Loop
- Reads `GOAL.md` — you write natural language tasks
- Hourly orchestration tick (configurable, I use 1h to save tokens)
- Iteratively decomposes work into kanban cards
- Pipeline v2 (complex tasks): aa-explorer [+ aa-researcher] → aa-planner → aa-plan-taste → aa-worker → aa-reviewer → aa-human-taste → aa-verifier → aa-goalkeeper
- Simple tasks skip pre-stages (explorer/researcher/planner/plan-taste)
- aa-orchestrator: dual role (cron heartbeat + Telegram chat / user comms)
- When stuck: escalates to you via `HELP_NEEDED.md` → Telegram (orchestrator handles dual)

### Human Taste Simulation
The `aa-human-taste` profile evaluates every output as a human would — detecting AI slop, unnatural language, design decisions that don't feel right. The orchestrator's inline taste-escalator decides when simulated taste is reliable and when it needs your real human judgment.

### Perpetual Continuous Audit
When all goals are complete, the system enters perpetual continuous audit:
- Rotates through ALL completed tasks, re-verifying deliverables
- Cycles progress through depths: VERIFICATION → QUALITY → DEEP IMPROVEMENT
- Finds issues → creates non-blocking polish cards (priority=0)
- Cycle complete → increments, resets, starts next immediately
- NEVER asks "what next?" — the audit is the default state
- Only NEW tasks from the user interrupt the loop

### Supporting Infrastructure
- **Orchestrator heartbeat** — drives the autonomous tick cycle on schedule
- **Watchdog** — alerts you when the agent needs help
- **Milestone reporter** — notifies you on phase transitions
- **Cleanup agent** — removes stale tasks, old temp files
- **Daily research** — searches the web for content matching your interests, writes summaries to `research/daily/`, and creates actionable proposals to `research/proposals/`
- **Research proposal reporter** — delivers research proposals for your review

## Quick Start

### Prerequisites
- Hermes Agent installed and configured
- At least one provider configured (strong model + cheap model)
- A messaging platform connected (Telegram recommended)
- Gateway running as systemd user service

### Installation

```bash
# Clone this repo
git clone https://github.com/Rybens92/autonomy-hermes-system ~/workspace/autonomy-hermes-system

# Ask your Hermes to install it:
hermes "Read ~/workspace/autonomy-hermes-system/GUIDE.html and install the autonomous agent system on my Hermes, adapting everything to my existing setup without breaking anything."
```

The `GUIDE.html` contains complete step-by-step instructions for AI agents — covering profile creation, cron job configuration, conflict detection, provider/model mapping, and verification.

### First Goal

```bash
mkdir -p ~/autonomous-agent
cat > ~/autonomous-agent/GOAL.md << 'EOF'
# Your Project Goal

Write what you want the autonomous agent to work on. Be specific but leave the
implementation details to the system.

Repozytorium: /path/to/your/project

## Tasks
1. First task
2. Second task
...

## Rules
- Code and docs in English
- Every change goes through the pipeline
EOF

echo "/path/to/your/project" > ~/autonomous-agent/PROJECT.md
```

### Watching It Work

```bash
# Check what's happening
hermes -p aa-orchestrator kanban list
cat ~/autonomous-agent/STATUS.md
cat ~/autonomous-agent/LOG.md

# Or check Telegram for milestone reports and help requests
```

## Customization

| What | Where | Default |
|------|-------|---------|
| Tick interval | `cron edit {heartbeat-id} --schedule "..."` | `0 * * * *` (hourly) |
| Messaging platform | Watchdog/milestone cron `--deliver` | auto-detected |
| Profile prefix | Profile config names | `aa-` |
| Strong model provider | `aa-orchestrator/config.yaml` | auto-mapped |
| Cheap model provider | `aa-worker/config.yaml` | auto-mapped |

## Important: What This Does NOT Do

- ❌ Does NOT modify Hermes source code
- ❌ Does NOT touch your existing profiles (uses `aa-` prefix)
- ❌ Does NOT require env vars (uses `.env` of the orchestrator profile)
- ❌ Does NOT depend on specific providers (auto-maps to what you have)
- ❌ Does NOT overwrite existing cron jobs (checks for conflicts)

## Token Cost

With hourly ticks and cheap models for workers/review/taste (strongest reasoning reserved for the orchestrator):

- **~30K strong-model tokens/day** for orchestration (at ~1 tick/hour + research)
- **~50K cheap-model tokens/day** for worker execution (varies with task volume)
- All supporting jobs (watchdog, cleanup, milestone): **0 LLM tokens** (no-agent scripts)

> The project creator uses STRONGEST reasoning model / CHEAPEST execution model.
> Your Hermes will choose models based on *your* configured providers.

## License

MIT — use it, fork it, ship it.
