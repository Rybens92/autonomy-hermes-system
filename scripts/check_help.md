# check_help.sh

## Purpose
Watchdog that monitors HELP_NEEDED.md and delivers help requests to the user's messaging platform. Runs as a no-agent cron job (zero LLM tokens).

## Key Logic

### Detection (line 4-5)
Checks if HELP_NEEDED.md exists and is non-empty (`[ -s "$HELP_FILE" ]`). If empty → silent exit (no output = no message delivered).

### Backup Before Delivery (line 6-8)
Creates a timestamped backup:`HELP_NEEDED.md.{timestamp}.sent` BEFORE attempting delivery. If the messaging platform is down or delivery fails, the help request survives.

### Delivery (line 10-14)
Outputs the help request with header, content, and timestamp. The cron job's stdout is delivered verbatim to the user's messaging platform.

### Clear On Success (line 17)
Uses `true > "$HELP_FILE"` to truncate only after the echo/output commands have executed. If the script crashes midway (e.g., file read error), the original is preserved.

## Modification Notes
- Change the backup directory if you want centralized audit logs
- The emoji `🚨` in the header can be changed/removed for your messaging platform
- If your messaging platform has character limits, add a truncation guard: `head -c 4000 "$HELP_FILE"`
