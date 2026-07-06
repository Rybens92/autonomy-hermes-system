# aa-thinker

## Role
Deep problem solver. Handles only the hardest tasks: system architecture, high-impact design decisions, non-standard debugging, multi-source research synthesis.

## Model Tier
**STRONGEST reasoning** — your most capable model. Highest token burn. Configured to max 200 turns with `reasoning_effort: xhigh`.

## Toolsets
- `terminal` — run commands, test hypotheses
- `file` — read/write project files
- `web` — search for solutions, research patterns
- `code_execution` — run Python scripts with Hermes tool access

## Key Behaviors
- Receives the hardest, most ambiguous tasks
- Full tool access for deep investigation
- Is a leaf worker — does NOT delegate (this is why it needs all tools)
- Must call `kanban_complete` after solving

## Modification Notes
- This is the most expensive profile in the system. Use sparingly — only for tasks where cheaper models would fail or produce low-quality output
- If you rarely use this, consider reducing `max_turns` to save costs on accidental invocation
- The toolsets MUST include `terminal`, `file`, `web`, `code_execution` — otherwise this profile is useless (can't investigate anything)
