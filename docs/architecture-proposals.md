# Architecture Proposals

> Ground-truth snapshot: 2026-07-14, Cycle 3 (Deep Improvement)
> 11 aa-* profiles, 7 active cron jobs (+ 2 disabled), aa-orchestrator dual role

## Current Architecture Analysis

### What's Working Well

**Pipeline v2 (pre-worker stages).** The explorer → researcher → planner → plan-taste → worker chain is well-structured. Parent/child dependency chaining in kanban ensures correct sequencing. Parallel execution of explorer + researcher saves wall-clock time. Simple tasks skip pre-stages via orchestrator routing.

**Single aa-orchestrator dual role.** GOAL.md goal 4 (aa-comms split) was correctly abandoned. A single orchestrator handling both cron orchestration and Telegram chat eliminates sync issues, gateway contention, and config drift between two profiles.

**Token-efficient design.** 4 of 7 active cron jobs are no-agent (script-only, 0 LLM tokens). Light audit from Cycle 3+ saves ~92% of audit tokens without quality loss. Health monitor is agent-based but designed as [SILENT]-on-healthy to avoid noise.

**Perpetual continuous audit.** The cycle-based audit loop (Verification → Quality → Deep Improvement) ensures deliverables don't stagnate. Cycle 3's light audit mode was a pragmatic response to observed token costs.

**System health monitor (recent).** A valuable addition that checks gateways, cron status, board state, profile existence, disk space, and file consistency. Already running every 3 hours.

**Kanban parent/child chaining.** Using `parents=[...]` on `kanban_create` to sequence pipeline stages is the right architectural pattern — the dispatcher handles sequencing naturally.

### What's Fragile

**Systemic profile unreliability.** From STATUS.md:
- aa-reviewer: 4 crashes (systemic) — tagged DO NOT ASSIGN
- aa-human-taste: unreliable (3/7 successes)
- aa-thinker: hallucinated an entire deliverable on first use

These profiles are still listed in the standard pipeline chain and used by the orchestrator. There is no feedback loop that tracks per-profile reliability over time — the orchestrator has no way to know "aa-reviewer has crashed 4 times this week" unless it reads STATUS.md.

**No error-rate tracking.** The orchestrator dispatches cards, profiles either succeed or fail, and the event is logged to kanban. But there is no aggregate view: which profiles fail most often, on which task types, with which models. Crashes are discoverable only by reading individual card events or the STATUS.md notes section (manual).

**Verifier workspace-vs-git gap (Pitfall #23).** Verifier checks workspace files (what the worker wrote to disk), not git commits. A worker that writes files but doesn't commit (or writes then reverts) passes verification. Multiple prior pipeline items in LOG.md flag this as a known weakness.

**Worker self-report vulnerability.** LOG.md documents multiple cases where workers claimed fabricated deliverables (Goal 4 chain, aa-comms profile creation). Pipeline reviewers verifier catches some but not all — the multi-agent debate isn't reliable against systematic fabrication.

**No backup or recovery.** There is zero backup infrastructure. If aa-orchestrator's config.yaml is corrupted, the orchestrator is down and there is no recovery path. Cron jobs, profile configs, and state files have no redundant copies.

**Config drift between live and git.** Despite the 1:1 sync mandate, there is no automated diff check. The system relies on the orchestrator remembering to sync. A stale repo means the universal blueprint diverges from the live system.

**Two disabled cron jobs with unclear resurrection path.** memory-observer and memory-consolidator have been disabled since early deployment. They accumulated ~66 and ~10 runs respectively. There is no documented condition under which they would be re-enabled.

**System health monitor uses STRONGEST model.** The health monitor runs procedural checks (run 5 commands, compare output). This could run on CHEAPEST tier with no quality loss, saving ~8 strong-model tokens per tick (3x daily = 24 strong-model ticks/week).

---

## Proposals

Ranked by **impact/effort ratio** (highest first).

---

### 1. Profile Reliability Dashboard

**Impact: HIGH | Effort: SMALL**

#### Problem
The orchestrator assigns profiles blindly — it has no aggregated view of which profiles fail, on which task types, or with which frequency. aa-reviewer has crashed 4 times systemically but continues to receive review cards. aa-human-taste succeeds only 3/7 times but appears in every pipeline. Without reliability data, the orchestrator cannot route around known failure patterns.

#### Proposed Solution
Add a lightweight reliability tracker as an extension to the existing system-health-monitor job. Create a `reliability_state.json` file that tracks per-profile success/failure counts and last-N-outcomes. Extend the health monitor's checks to read this file and flag profiles crossing failure thresholds.

#### Implementation Sketch
1. Create `~/.hermes/profiles/aa-orchestrator/scripts/reliability_tracker.sh`:
   - Reads kanban board to find completed/failed tasks per profile
   - Updates `reliability_state.json` with running counts
   - Alerts (via stdout, caught by health monitor) when a profile exceeds failure threshold
2. Extend system-health-monitor prompt to include a "Profile reliability" check that reads `reliability_state.json`
3. (Optional) Create `aa-reliability-observer` profile with `file` + `kanban` toolsets for richer reliability analysis

#### Impact
- Orchestrator can route around known-unreliable profiles
- Early warning when a profile starts degrading
- Data-driven decisions on prompt improvements or toolset adjustments
- Reduces wasted tokens on doomed pipeline stages

#### Risks
- Two profiles plus script changes add surface area
- Reliability data could be stale if scripts don't run frequently enough
- Might detect correlation (profile X always gets hard tasks) instead of causation

#### Effort: Small (1-2 kanban cards: 1 script + 1 prompt extension)

---

### 2. Git-anchored Verification Gate

**Impact: MEDIUM | Effort: MEDIUM**

#### Problem
Workers can pass the verification pipeline without ever committing to git. Pitfall #23 (from LOG.md Cycle 3 audit) documents this gap: "Verifier checks workspace files, not committed artifacts" — enabling self-report fabrication through the entire review chain. Multiple prior pipeline runs showed workers claiming deliverables that didn't exist in committed form.

#### Proposed Solution
Add a "git commit verification" step to the aa-verifier prompt and a corresponding "commit requirement" to the aa-worker prompt. The worker must commit meaningful changes before calling `kanban_complete`. The verifier explicitly checks that commits exist and contain the claimed deliverables.

#### Implementation Sketch
1. **aa-worker prompt change (small):** Add instruction: "Before calling kanban_complete, git add + commit your changes with a descriptive message. If the workspace has no .git, document this in the summary."
2. **aa-verifier prompt change (small):** Add instruction: "Verify that claimed deliverables are committed to git. Run `git log --oneline -5` and check that commits contain the correct files. If the workspace has no .git, note this."
3. **aa-reviewer prompt change (optional):** Add: "Check that work is committed, not just sitting on disk."

#### Impact
- Closes a known gap that has allowed fabricated deliverables through the pipeline
- Makes the audit trail complete — every deliverable links to a commit
- Enables downstream verification to check actual git state, not workspace state
- Smallest change surface (prompt edits only, no new profiles or cron jobs)

#### Risks
- Some task workspaces may not be git repos (kanban scratch workspaces). The changes should handle this gracefully — note the absence, don't fail.
- Commit-first workflow adds friction for exploratory/debugging tasks — but the recommendation already says "If you are inside a git repository, commit meaningful changes as you go"

#### Effort: Medium (3 config.yaml prompt edits across aa-worker, aa-verifier, optionally aa-reviewer)

---

### 3. Config Backup & Recovery System

**Impact: HIGH | Effort: LARGE**

#### Problem
There is no backup for any part of the autonomous system. A config.yaml corruption, accidental profile deletion, or botched cron job update would require manual reconstruction. The orchestrator cannot recover itself because it IS the thing that would need recovery.

#### Proposed Solution
Create a backup profile (`aa-backup-agent`) and a periodic cron job that snapshots all critical system state to a backup directory. Include a restoration procedure doc.

#### Implementation Sketch
1. Create new profile `aa-backup-agent` (toolsets: `file`, `terminal`, `kanban`, no web/model needed):
   - system_prompt: reads all profile config.yaml files, cron jobs.json, state files (GOAL.md, STATUS.md, LOG.md, HELP_NEEDED.md, PROJECT.md), and scripts → copies to `~/autonomous-agent/backups/YYYY-MM-DD/`
   - Runs on CHEAPEST tier (no reasoning needed for cp/rsync operations)
2. Add cron job under aa-orchestrator:
   ```json
   {
     "name": "config-backup-agent",
     "schedule": "0 4 * * *",  // daily at 04:00
     "prompt": "Backup all profile configs, cron jobs, scripts, and state files to ~/autonomous-agent/backups/<date>/. Create a manifest listing what was backed up.",
     "skills": [],
     "deliver": "local",  // silent; only alert on failure
     "no_agent": false
   }
   ```
3. Write `docs/recovery-procedure.md` with step-by-step restoration instructions
4. Update `~/.hermes/profiles/aa-orchestrator/scripts/recovery_check.sh` (optional — verify backup exists and is not stale)

#### Impact
- Complete system recovery from a known-good snapshot
- Daily cadence means at most 24h of work lost
- Backup agent uses cheap model tier → negligible token cost
- Restoration procedure exists before it's needed (proactive, not reactive)

#### Risks
- Daily backup with an LLM agent costs ~500 tokens per run. Could make it no-agent with a script, but scripts can't write files across profiles (cross-profile guard).
- Backup agent has terminal access — must ensure it only reads, never writes live configs
- Recovery procedure may go stale if profile count changes without doc update
- Verification: how do you test recovery without actually breaking something?

#### Effort: Large (1 new profile + 1 cron job + 1 doc file + 1 optional script)

---

### 4. Orchestrator Decision Audit Trail

**Impact: MEDIUM | Effort: SMALL**

#### Problem
The orchestrator makes hundreds of decisions per week (create cards, route profiles, audit tasks), but only major decisions get logged to LOG.md. There is no structured record of:
- Every card the orchestrator created and why
- Every route choice (why Pre-worker stages A vs B)
- Every pipeline result (which stages passed/failed)
- Every audit finding

This makes debugging poor orchestrator decisions difficult — you'd have to reconstruct intent from LOG.md prose and kanban card states.

#### Proposed Solution
Create a structured decision log as a JSON file (`decision_log.json`) that the orchestrator appends to on every tick. Each entry captures: tick time, decision type, card IDs involved, routing choices, and outcome. A companion script can then summarize or query this log.

#### Implementation Sketch
1. Add to aa-orchestrator system prompt section on logging: "After each tick, append a JSON entry to `~/autonomous-agent/decision_log.json` with: timestamp, decision_type (create_card/audit/route/update_state), card_ids, rationale, result. Keep entries compact — one line per entry."
2. Add a small cron job (no-agent, script-only, every 6h) that truncates the log if it exceeds 5000 lines by removing the oldest entries
3. The log is an append-only JSONL file — one JSON object per line

#### Impact
- Full audit trail of orchestrator decisions
- Enables retroactive analysis of routing patterns, error rates, audit depth effectiveness
- Small token overhead (a few lines per tick appended to LOG.md prose)
- No new profile needed — purely a prompt change + one script

#### Risks
- JSONL file grows unbounded without the truncation script
- Orchestrator might format entries inconsistently
- Adds ~2-3 lines to each orchestration tick output

#### Effort: Small (1 prompt edit + 1 optional script)

---

### 5. Memory Pipeline Re-activation (Re-enable Memory Observer + Consolidator)

**Impact: LOW-MEDIUM | Effort: MEDIUM**

#### Problem
Two cron jobs (memory-observer at XX:05, memory-consolidator at 00:00/12:00) have been disabled since early deployment. They were designed to capture system behavior patterns and merge them into topic files. The current system has no automated learning from past decisions — every audit tick is a fresh look at the same artifacts, not informed by cumulative system knowledge.

The reason for disabling was "observer writes to unread files" — nobody consumed the observations. But the system now has more profiles, longer runtime history, and concrete reliability problems (aa-reviewer crashes) that a memory system could help diagnose.

#### Proposed Solution
Re-enable the memory pipeline with a narrow, concrete consumer: the Profile Reliability Dashboard (Proposal 1 above). Memory observations feed the reliability tracker. This gives the observer+consolidator a known consumer and a clear value proposition.

#### Implementation Sketch
1. Update memory-observer prompt to focus on: profile crash patterns, failed task types, frequency of [SILENT] responses, HELP_NEEDED.md usage patterns
2. Update memory-consolidator to produce: per-profile reliability reports, task-type-to-failure correlations, weekly trend summaries
3. Wire the consolidator output into the reliability tracker script (Proposal 1)
4. Re-enable both cron jobs in jobs.json

#### Impact
- Turns dark data (crashes, failures, patterns) into actionable insight
- Gives the memory pipeline a concrete consumer with measurable value
- Adds ~12,800 tokens/day in strong-model tokens (as documented in cron-jobs.md)
- Enables future ML-driven improvements (auto-tune routing weights based on pattern data)

#### Risks
- Token cost is not negligible (~12,800/day = ~$0.25-0.50/day at typical reasoning model pricing)
- Observations might be noise-dominated without careful prompt design
- Feedback loop risk: if observer observes observer behavior, you get infinite recursion
- Requires prompt tuning iteration — the original prompts were generic (`.`) so the whole behavioral design is fresh work

#### Effort: Medium (2 prompt rewrites + wire-up to Proposals 1 scripts)

---

## Recommendation Order

| Rank | Proposal | Impact | Effort | Rationale |
|------|----------|--------|--------|-----------|
| 1 | Profile Reliability Dashboard | HIGH | SMALL | Closes the biggest operational gap — flying blind on profile health. Low effort, immediate value. |
| 2 | Git-anchored Verification Gate | MEDIUM | MEDIUM | Fixes a known vulnerability (Pitfall #23) with prompt-only changes. No new infrastructure. |
| 3 | Orchestrator Decision Audit Trail | MEDIUM | SMALL | Cheap insurance for debugging orchestrator behavior. Tiny token overhead. |
| 4 | Config Backup & Recovery System | HIGH | LARGE | High impact but larger scope (new profile, cron job, doc). Important insurance but not urgent if system is stable. |
| 5 | Memory Pipeline Re-activation | LOW-MEDIUM | MEDIUM | Most value only after Proposal 1 exists (to consume the data). Token cost requires clear ROI — defer until reliability tracking proves its value. |

**Do first:** Proposal 1 (Reliability Dashboard) — get visibility into profile health.
**Do second:** Proposal 2 (Git-anchored Verification) — close the workspace-vs-git gap.
**Do third:** Proposal 3 (Decision Audit Trail) — low-hanging fruit for debugging.
**Defer:** Proposal 4 (Backup) — critical but large; plan for next cycle.
**Defer pending Proposal 1 success:** Proposal 5 (Memory Pipeline) — only if reliability data shows value worth the token cost.

## Appendix: Data Sources

All proposals are grounded in actual system state as of 2026-07-14:

- **Profile list**: 11 aa-* profiles confirmed live
- **Cron jobs**: 7 active + 2 disabled confirmed in jobs.json
- **Crash data**: aa-reviewer (4 crashes, systemic), aa-human-taste (3/7 unreliable), aa-thinker (1 hallucination) — from STATUS.md
- **Pitfalls**: Pitfall #23 (workspace-vs-git gap), Pitfall #26 (gateway switching structurally impossible) — from LOG.md
- **Pipeline**: v2 with pre-worker stages for complex tasks, simple tasks skip to worker — documented in GOAL.md
- **Architecture decision history**: Goal 4 (aa-comms split) abandoned, single aa-orchestrator dual-role confirmed permanent — from GOAL.md Section 4
