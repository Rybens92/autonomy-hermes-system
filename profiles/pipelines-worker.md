# aa-worker

## Role
Primary task executor. Receives kanban cards from the orchestrator and produces concrete output: code changes, test results, documentation, research findings.

## Model Tier
**CHEAPEST capable** — this profile runs frequently, so use your most cost-efficient model that can still produce quality work. Configured to max 100 turns.

## Toolsets
- `terminal` — run commands, execute tests, install dependencies
- `file` — read/write project files
- `web` — search documentation, research APIs
- `code_execution` — run Python scripts with Hermes tool access

## Key Behaviors
- Receives detailed task instructions in the kanban card body
- Works in the directory set by the kanban dispatcher (`HERMES_KANBAN_WORKSPACE`)
- Must call `kanban_complete` before exiting — failure to do so causes a protocol violation crash
- Is a leaf worker — does NOT delegate further

## Modification Notes
- Increase `max_turns` if working on large codebases with many files to modify
- Add `reasoning_effort: high` if the cheap model tends to cut corners
- The working directory (`terminal.cwd`) is a FALLBACK — the kanban dispatcher overrides it
