# System Information

Collect macOS system information and display as a clean table.

## Data Collection

Run the following commands in parallel where possible.

### macOS Version

```bash
sw_vers
```

### Hardware Model

```bash
sysctl -n hw.model
system_profiler SPHardwareDataType | grep -E "Model Name|Chip|Total Number of Cores|Memory"
```

### CPU Usage

```bash
top -l 1 -n 0 | grep "CPU usage"
```

### Memory Usage

```bash
sysctl -n hw.memsize
vm_stat
```

Conversion: parse the page size from the first line of `vm_stat` output (e.g., "page size of 16384 bytes"). Multiply page counts by this parsed page size — do NOT hardcode 4096 (Apple Silicon uses 16384). Calculate used = total - (free_pages × page_size). Do not subtract inactive pages — inactive memory is reclaimable but still in use. Show used/total in GB (divide by 1073741824).

### Storage Overview

```bash
diskutil info / | grep -E "Container (Free|Total) Space"
```

### Battery Health

```bash
system_profiler SPPowerDataType | grep -E "Cycle Count|Condition|Maximum Capacity"
```

Skip battery on desktops: run `system_profiler SPPowerDataType` and check if the output contains "Battery Information". If it does, include battery stats. If not (desktops like iMac, Mac Pro, Mac mini, Mac Studio), omit battery entirely.

### Uptime

```bash
uptime
```

### Network

First detect the active network service:

```bash
networksetup -listallnetworkservices
```

Then query the first active service (try Wi-Fi first, fall back to Ethernet or other available services). If no service is available, show "No active network connection".

```bash
networksetup -getinfo "Wi-Fi"
# If Wi-Fi fails or doesn't exist, try:
# networksetup -getinfo "Ethernet"
```

## Output Format

Present all results in a single table:

```
| Item            | Value                        |
|-----------------|------------------------------|
| macOS           | ...                          |
| Model           | ...                          |
| Chip            | ...                          |
| Cores           | ...                          |
| Memory          | X.X GB used / X.X GB total   |
| CPU Usage       | user% user, sys% sys, idle%  |
| Storage         | X.X GB free / X.X GB total   |
| Battery         | Condition, Cycle Count, Max% (omit this row entirely on desktops) |
| Uptime          | ...                          |
| Network (Wi-Fi) | IP, subnet, router           |
```

### Health Summary

End with a one-line health summary. Flag concerns if any:
- CPU usage > 80% (user + sys)
- Free memory < 2 GB
- Battery condition is not "Normal"
- Free storage < 10% of total

If nothing is flagged, report "All systems healthy."

## Summary Mode

When invoked in summary mode (from full checkup), show only these items:
- macOS version
- Chip
- Memory (used/total)
- Storage (free space)
- Battery condition (if laptop)
- Uptime

Skip CPU usage and network details. Use the same table format. Still include the one-line health summary.
