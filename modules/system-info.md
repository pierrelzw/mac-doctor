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

Conversion: multiply page counts from `vm_stat` by 4096 to get bytes. Calculate used = total - (free * 4096) - (inactive * 4096). Show used/total in GB (divide by 1073741824).

### Storage Overview

```bash
diskutil info / | grep -E "Container (Free|Total) Space"
```

### Battery Health

```bash
system_profiler SPPowerDataType | grep -E "Cycle Count|Condition|Maximum Capacity"
```

Skip battery on desktops: if `sysctl -n hw.model` contains "Mac" but not "Book" (iMac, Mac Pro, Mac mini, Mac Studio), omit battery entirely.

### Uptime

```bash
uptime
```

### Network

```bash
networksetup -getinfo Wi-Fi
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
| Battery         | Condition, Cycle Count, Max% |
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
