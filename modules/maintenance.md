# Maintenance Module

macOS diagnostics and repair operations.

## Part 1: Diagnostics

Run these checks and report results:

### Disk Health
Command: `diskutil verifyVolume /`
Report pass/fail and any errors found.

### System Integrity Protection (SIP)
Command: `csrutil status`
Flag if disabled — this is unusual and potentially risky.

### Login Items
Command: `osascript -e 'tell application "System Events" to get the name of every login item'`
List all items. Flag if count exceeds 10.
Note: On macOS 13+ (Ventura), this command may fail with error -10827 if the terminal lacks Automation permission. If it fails, tell the user to check System Settings > General > Login Items directly.

### Recent Crash Logs
Command: `ls -lt ~/Library/Logs/DiagnosticReports/ 2>/dev/null | head -10`
Show recent crashes. Highlight any from the last 24 hours.

### System Log Errors (last hour)
Command: `/usr/bin/log show --last 1h --predicate 'eventType == logEvent AND logType == error' --style compact 2>/dev/null | tail -20`
Summarize the patterns in these recent error lines. Note: this is a sample of the most recent errors, not a complete count — do not fabricate total counts.

## Part 2: Repairs

Present available repairs as a numbered menu. Execute ONLY after user selects and confirms each one individually.

5 repair operations:
1. Flush DNS Cache — `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder` — Low risk — useful for DNS resolution problems
2. Rebuild Spotlight Index — `sudo mdutil -E /` — Low risk but time-consuming (reindexing may take hours, Spotlight search will be unavailable during this time, and CPU/disk usage will be elevated) — useful when search not finding files. Before offering, check if reindexing is already in progress: `mdutil -s /`.
3. Rebuild Launch Services Database — `/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -kill -r -domain local -domain system -domain user` — Low risk — useful when wrong app opens for file types
4. Clear Font Cache — `sudo atsutil databases -remove` — Low risk, may require restart — useful for font rendering issues. **Only available on macOS <= 13 (Ventura).** On macOS 14 (Sonoma) and later, `atsutil` is removed. Check `sw_vers -productVersion` before offering this option.
5. Repair Disk Permissions — `diskutil repairPermissions /` — Low risk — ONLY works on macOS <= 10.11 (El Capitan and earlier). Apple removed the underlying mechanism in 10.12 Sierra. Check `sw_vers -productVersion` before offering this option. On Sierra+ this is handled automatically by the system and the command is a no-op.

## Rules
- Every sudo command requires explicit user confirmation before execution
- Execute repairs one at a time, report result before offering next
- After each repair, explain what was done and whether it succeeded
- If a command fails, report the error clearly and suggest next steps

## Summary Mode
When invoked in summary mode, run only these lightweight diagnostics from Part 1: Disk Health, SIP status, and Recent Crash Logs. Skip System Log Errors (slow) and Login Items (may need permissions). Skip Part 2 (Repairs) entirely. Report a one-line health status per check.
