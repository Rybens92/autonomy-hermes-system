# milestone_reporter.sh

## Purpose
Detects STATUS.md changes and reports phase transitions to the user's messaging platform. Runs as no-agent (zero LLM tokens). Deduplicates via hash comparison of grep-filtered content — not the full file.

## Key Logic

### Hash Detection (lines 9-14)
Computes MD5 hash of **grep-filtered lines only** — not the full STATUS.md. The filtering selects lines matching milestone markers:

```bash
grep -E "\[MILESTONE\]|\[BLOCKED\]|\[DONE\]|\[REVIEW_FAILED\]|Phase [0-9]" "$STATUS" | tail -5
```

Only the last 5 matching lines are hashed. The hash is stored in `.last_milestone_hash` (line 7). If the hash matches the previous run → silent exit (no change). If different → proceeds to output.

### Status Check (lines 17-19)
If `CURRENT_SIG == LAST_SIG`, the script exits silently (`exit 0`). No output = no message sent.

### Hash Storage (line 23)
Saves the new hash to the hash file:
```bash
echo "$CURRENT_SIG" > "$HASH_FILE"
```

### Output (lines 26-31)
Prints a filtered status update with a hardcoded project header:
```
|📊 {PROJECT_NAME} Status Update:

<grep-filtered lines: Phase, Status, DONE, BLOCKED, MILESTONE, REVIEW_FAILED — last 5>

Ostatni tick: <value from STATUS.md>
```

The project name is now set via the `{PROJECT_NAME}` placeholder (line 29). Replace `{PROJECT_NAME}` with your actual project name, or set `PROJECT_NAME` as an environment variable in the cron job definition.

## Customization Points
- **Project name**: Replace `{PROJECT_NAME}` placeholder on line 29 with your project name, or set `PROJECT_NAME` as an environment variable
- **Hash file**: `.last_milestone_hash` (line 7) — stored in the scripts directory. Change the path if you want it elsewhere
- **Hash algorithm**: Uses `md5sum`. Replace with `sha256sum` if your system doesn't have md5sum
- **Filter pattern**: Adjust the grep patterns on line 10 to match your STATUS.md format
- **Line count**: Change `tail -5` to a different number to include more/fewer status markers
