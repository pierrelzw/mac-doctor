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

### Recent Crash Logs
Command: `ls -lt ~/Library/Logs/DiagnosticReports/ 2>/dev/null | head -10`
Show recent crashes. Highlight any from the last 24 hours.

### System Log Errors (last hour)
Command: `log show --last 1h --predicate 'eventType == logEvent AND logType == error' --style compact | tail -20`
Summarize: count of errors, top recurring messages.

## Part 2: Repairs

Present available repairs as a numbered menu. Execute ONLY after user selects and confirms each one individually.

5 repair operations:
1. Flush DNS Cache — `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder` — Low risk — useful for DNS resolution problems
2. Rebuild Spotlight Index — `sudo mdutil -E /` — Low risk but time-consuming — useful when search not finding files
3. Rebuild Launch Services Database — `/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -kill -r -domain local -domain system -domain user` — Low risk — useful when wrong app opens for file types
4. Clear Font Cache — `sudo atsutil databases -remove` — Low risk, may require restart — useful for font rendering issues
5. Repair Disk Permissions — `diskutil repairPermissions /` — Low risk — ONLY works on macOS < 10.15 (Mojave and earlier). Check `sw_vers` before offering this option. On Catalina+ this is handled automatically.

## Rules
- Every sudo command requires explicit user confirmation before execution
- Execute repairs one at a time, report result before offering next
- After each repair, explain what was done and whether it succeeded
- If a command fails, report the error clearly and suggest next steps

## Summary Mode
When invoked in summary mode, run only Part 1 (Diagnostics). Skip Part 2 (Repairs) entirely. Report a one-line health status per check.
