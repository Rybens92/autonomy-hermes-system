# milestone_reporter.sh

## Purpose
Detects STATUS.md changes and reports phase transitions to the user's messaging platform. Runs as no-agent (zero LLM tokens). Deduplicates via hash comparison.

## Key Logic

### Hash Detection (line 6-9)
Computes MD5 hash of STATUS.md. Compares against a stored hash in `.milestone_hash`. If identical → silent exit (no change). If different → report the new status and update the stored hash.

### Output (line 11-14)
Prints the full STATUS.md content with a header. Because stdout is delivered verbatim, the user sees the complete new status.

## Modification Notes
- Use `sha256sum` instead of `md5sum` if your system doesn't have md5sum
- The hash file `.milestone_hash` is stored in the scripts directory. Change if you want it elsewhere.
- Add filtering: only report certain status fields (e.g., grep "Phase:" from STATUS.md instead of full content)
