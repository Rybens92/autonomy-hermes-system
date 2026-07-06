# aa-goalkeeper

## Role
Global goal achievement check. After all pipeline phases complete, this profile evaluates whether the user's stated goal in GOAL.md has actually been achieved.

## Model Tier
**BALANCED** — needs good comprehension but works on a structured assessment.

## Toolsets
- `file` — read GOAL.md, LOG.md, STATUS.md, project files
- `web` — verify external references if needed

## Key Behaviors
1. Reads GOAL.md — what was the user's goal?
2. Reads LOG.md — what was actually done?
3. Reads STATUS.md — what's the current state?
4. Verifies artifacts against specification
5. Outputs SCORE 0-10 + VERDICT (GOAL_ACHIEVED | IN_PROGRESS | FAILED) + EVIDENCE
6. If incomplete → lists exact remaining work
7. Must call `kanban_complete` after assessment

## Modification Notes
- The verdict structure can be extended with custom statuses (e.g., PARTIALLY_ACHIEVED)
- If the project is large, increase `max_turns` to allow reading all artifacts
