# Disk Cleanup Module

Scan disk usage, identify cleanable items, and help the user reclaim space safely.

## Phase 1: Scan

### 1.1 Check available space

Use `diskutil info /` to get accurate free space. Record free space as the baseline.

```bash
diskutil info / | grep -E "Container (Free|Total) Space|Volume (Free|Total) Space"
```

On APFS volumes, use `Container Free Space` / `Container Total Space`. On HFS+ volumes (older Macs), these fields are absent — fall back to `Volume Free Space` / `Volume Total Space` instead.

### 1.2 Scan large directories

Run these in parallel to find space hogs:

```bash
# Top-level visible directories
du -sh ~/*/  2>/dev/null | sort -rh | head -15

# Hidden directories (exclude . and ..)
du -sh ~/.[^.]* ~/..?*  2>/dev/null | sort -rh | head -20

# Library caches
du -sh ~/Library/Caches/*/  2>/dev/null | sort -rh | head -15

# Application Support
du -sh ~/Library/Application\ Support/*/  2>/dev/null | sort -rh | head -15
```

### 1.3 Drill into large items from 1.2

For each large directory found in 1.2, drill one level deeper to identify what's inside:

```bash
# For each large item from 1.2, drill into subdirectories and large files
du -sh <large-dir>/* 2>/dev/null | sort -rh | head -10
```

Additionally, scan these locations that 1.2 may miss:

```bash
# Containers (sandboxed apps like WeChat, QQ, etc.)
du -sh ~/Library/Containers/*/ 2>/dev/null | sort -rh | head -15

# Group Containers (shared data for sandboxed apps)
du -sh ~/Library/Group\ Containers/*/ 2>/dev/null | sort -rh | head -15

# Trash
du -sh ~/.Trash/ 2>/dev/null

# iOS device backups (list individual backups for user to choose)
ls -1 ~/Library/Application\ Support/MobileSync/Backup/ 2>/dev/null | while read uuid; do
  du -sh ~/Library/Application\ Support/MobileSync/Backup/"$uuid" 2>/dev/null
done | sort -rh
```

Check for APFS local Time Machine snapshots (these can consume significant space and are not visible to `du`):

```bash
tmutil listlocalsnapshots / 2>/dev/null
```

If snapshots exist, report the count and suggest `sudo tmutil deletelocalsnapshots <date>` as a cleanup option (requires user confirmation).

For items that need CLI commands for detailed breakdown, check if the tool exists first:

```bash
# Only run these if the command exists
command -v ollama &>/dev/null && ollama list
command -v docker &>/dev/null && docker system df 2>/dev/null
command -v pip3 &>/dev/null && pip3 cache info 2>/dev/null
```

Analyze all results and identify what each large item is. Use your knowledge of macOS app data locations to determine the app name, whether it's a cache (auto-rebuilds) or user data (irreplaceable), and whether the app is still installed.

### 1.4 Detect orphaned app data

Look for Application Support and Containers directories where the app is no longer installed:

```bash
# Check Application Support for orphaned app data
for dir in ~/Library/Application\ Support/*/; do
  name=$(basename "$dir")
  if ! find /Applications ~/Applications /System/Applications -maxdepth 2 -iname "*${name}*.app" -print -quit 2>/dev/null | grep -q .; then
    size=$(du -sh "$dir" 2>/dev/null | cut -f1)
    echo "$size	$name (possibly orphaned)"
  fi
done | sort -rh | head -10

# Check Containers for orphaned sandboxed app data
for dir in ~/Library/Containers/*/; do
  name=$(basename "$dir")
  if ! find /Applications ~/Applications -maxdepth 2 -iname "*${name}*.app" -print -quit 2>/dev/null | grep -q .; then
    size=$(du -sh "$dir" 2>/dev/null | cut -f1)
    echo "$size	$name (possibly orphaned)"
  fi
done | sort -rh | head -10
```

This is a heuristic — app bundle names don't always match support directory names (e.g., `BraveSoftware` vs `Brave Browser.app`). Present results as "possibly orphaned" and let the user decide. When uncertain, check `mdls -name kMDItemCFBundleIdentifier` on the directory to help identify the app.

## Phase 2: Report

Present findings in a table grouped by safety level. Always show size and what it is. Only list items that were actually found during scanning.

### Category: Safe to delete (caches that auto-rebuild)

Items in `~/Library/Caches/`, package manager caches, build caches, transfer caches, logs, and Trash. These regenerate automatically when the app runs again. Use your knowledge to classify — a directory named `Cache`, `Caches`, `tmp`, `logs`, `DerivedData`, `_cacache`, or similar is almost always safe.

### Category: Review before deleting (data won't auto-rebuild but may not be needed)

Downloaded content the user chose to cache locally: AI models, offline music/podcasts, game assets, old iOS device backups, Docker images. The data can be re-downloaded, but the user may want to keep some of it.

### Category: Caution (user data, could be important)

Chat history, recordings, project files, documents, mail, photos — anything that represents the user's own content rather than a cache of externally available data. These should never be suggested for deletion without clear justification.

## Phase 3: Clean

Wait for user to choose what to clean. Then:

1. Show the user the exact commands that will be run and the estimated space to be reclaimed
2. Wait for final confirmation before executing
3. Execute cleanup commands for selected items
4. After cleanup, re-check space with `diskutil info /`
5. Report before/after comparison: how much free space was gained

### Cleanup approach

For each item the user selects, construct the appropriate cleanup command based on what the item is:

- **Caches in `~/Library/Caches/`**: `rm -rf <path>` (safe, they rebuild)
- **Package manager caches**: use the tool's own purge command when available (e.g., `pip3 cache purge`, `npm cache clean --force`, `conda clean --all -y`), fall back to `rm -rf` for the cache directory
- **AI models**: use the tool's removal command (e.g., `ollama rm <model>`) so the registry stays consistent
- **Docker**: `docker system prune` (with appropriate flags based on what user wants to remove)
- **Trash**: `rm -rf ~/.Trash/* ~/.Trash/.*` (include dotfiles)
- **Logs**: `rm -rf ~/Library/Logs/*`
- **User data** (chat files, recordings, project files): never delete via CLI — show the path and tell the user to review in Finder

Notes:
- Browser caches may require the browser to be quit first due to file locks.
- For iOS device backups, show the path and let the user delete specific backups in Finder.

## Important rules

- Never delete anything without user confirmation — always confirm each category individually, even if user says "delete all"
- Always report the before/after free space delta using `diskutil info /`
- When unsure if something is safe to delete, research it first (check if the app is still installed, if the command/binary exists, etc.)
- For items you're not confident about, say so — don't guess
- Do not use `df -h` for free space reporting — it can be misleading on APFS. Use `diskutil info /` instead.

## Summary Mode

When invoked in summary mode (from full checkup), run only Phase 1 steps 1.1 (check available space) and 1.2 (scan large directories). Report total free space and top 5 largest directories. Skip drill-downs (1.3), orphan detection (1.4), and Phases 2-3.
