# Autonomy Hermes System

Turn your [Hermes Agent](https://github.com/nousresearch/hermes-agent) into a fully autonomous agent that works on multi-phase projects, reports to you via your messaging platform, simulates human taste in decision-making, and asks you for direction when it's stuck — all without modifying a single line of Hermes source code.

## What This Is

A drop-in configuration layer that sits on top of your existing Hermes installation. It uses **only built-in Hermes features**:

- **Cron jobs** — scheduled autonomous ticks (hourly by default)
- **Kanban board** — task dispatch across specialized worker profiles
- **Profiles** — different models/toolsets for different roles
- **Scripts** — context injection, watchdog, cleanup, reporting
- **Gateway** — messaging platform delivery (Telegram, Discord, Slack…)

## Architecture at a Glance

```
GOAL.md              user writes WHAT to do
   │
   ▼
cron: heartbeat ──── state_context.sh injects context
   │                       │
   ▼                       ▼
aa-orchestrator ──── reads state, creates kanban cards
   │                                           │
   │    ┌──────────────────────────────────────┘
   │    ▼
   │  kanban dispatcher ─► aa-worker      (cheapest capable model)
   │                         aa-reviewer   (balanced model)
   │                         aa-human-taste (AI slop detection)
   │                         aa-verifier   (fact/hallucination check)
   │                         aa-escalator  (meta: ask user?)
   │                         aa-thinker    (strongest reasoning — architecture)
   │                         aa-goalkeeper (global goal check)
   │
   ▼
Gate decisions: run pipeline ──► approve ──► next phase
                  │
                  ▼
            blocker ──► HELP_NEEDED.md ──► watchdog ──► Telegram 🚨
```

## Features

### Core Loop
- Reads `GOAL.md` — you write natural language tasks
- Hourly orchestration tick (configurable, I use 1h to save tokens)
- Iteratively decomposes work into kanban cards
- Pipeline: worker → reviewer → human-taste → verifier
- When stuck: escalates to you via `HELP_NEEDED.md` → Telegram

### Human Taste Simulation
The `aa-human-taste` profile evaluates every output as a human would — detecting AI slop, unnatural language, design decisions that don't feel right. The `aa-escalator` profile decides when simulated taste is reliable and when it needs your real human judgment.

### Self-Audit
When all goals are complete, the system doesn't just sit idle. It:
- Re-reads its own work (LOG.md, STATUS.md)
- Evaluates quality from a human perspective
- Creates polish cards for things to improve
- Asks you: "Done. What should I work on next? Based on your interests: [proposals]"

### Supporting Infrastructure
- **Memory observer** — captures system behavior patterns
- **Memory consolidator** — merges observations into topics
- **Watchdog** — alerts you when the agent needs help
- **Milestone reporter** — notifies you on phase transitions
- **Cleanup agent** — removes stale tasks, old temp files
- **Daily research** — finds relevant papers/posts based on your interests

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

> The project creator uses GLM-5.2 for orchestration and DeepSeek v4 for execution.
> Your Hermes will choose models based on *your* configured providers.

## License

MIT — use it, fork it, ship it.
