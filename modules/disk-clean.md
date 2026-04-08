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

# Hidden directories
du -sh ~/.*/  2>/dev/null | sort -rh | head -20

# Library caches
du -sh ~/Library/Caches/*/  2>/dev/null | sort -rh | head -15

# Application Support
du -sh ~/Library/Application\ Support/*/  2>/dev/null | sort -rh | head -15
```

### 1.3 Drill into known space hogs

Check these common large items in parallel:

- **Ollama models**: `ollama list`
- **HuggingFace cache**: `du -sh ~/.cache/huggingface/hub/*/ ~/.cache/huggingface/xet/ 2>/dev/null | sort -rh`
- **npm cache**: `du -sh ~/.npm/_cacache/ 2>/dev/null`
- **pip cache**: `pip3 cache info 2>/dev/null`
- **conda cache**: `du -sh ~/miniconda3/pkgs/ 2>/dev/null`
- **Homebrew cache**: `du -sh ~/Library/Caches/Homebrew/ 2>/dev/null`

### 1.4 Detect orphaned app data

Look for Application Support directories where the app is no longer installed:

```bash
# List app support dirs and check if corresponding .app exists
for dir in ~/Library/Application\ Support/*/; do
  name=$(basename "$dir")
  if ! ls /Applications/ ~/Applications/ 2>/dev/null | grep -qi "$name"; then
    size=$(du -sh "$dir" 2>/dev/null | cut -f1)
    echo "$size	$name (possibly orphaned)"
  fi
done 2>/dev/null | sort -rh | head -10
```

This is a heuristic — app names don't always match directory names. Present results as "possibly orphaned" and let the user decide.

### 1.5 Check iCloud status

```bash
# Check if optimize storage is enabled
optimize=$(defaults read com.apple.bird optimize-storage 2>/dev/null)
echo "Optimize Mac Storage: $optimize (1=enabled, 0=disabled)"

# Check for large locally-downloaded iCloud files that could be evicted
# Files with no .icloud extension are downloaded locally
find ~/Documents -xdev -maxdepth 3 -size +500M -not -name "*.icloud" 2>/dev/null | head -10
```

Note: there is no CLI command to evict iCloud files. If suggesting iCloud eviction, tell the user to right-click the folder in Finder and uncheck "Keep Downloaded".

## Phase 2: Report

Present findings in a table grouped by safety level. Always show size and what it is.

### Category: Safe to delete (caches that auto-rebuild)

These can be deleted without any data loss:

- pip cache (`pip3 cache purge`)
- npm cache (`npm cache clean --force`)
- Homebrew cache (`rm -rf ~/Library/Caches/Homebrew/`)
- conda cache (`conda clean --all -y`)
- HuggingFace xet transfer cache (`~/.cache/huggingface/xet/`)
- Browser caches (Google, Firefox, Edge, Brave in `~/Library/Caches/`)
- App update caches (\*.ShipIt directories, \*-updater directories)
- Playwright browsers (`~/Library/Caches/ms-playwright/`)

### Category: Review before deleting (data won't auto-rebuild)

- AI models (Ollama, HuggingFace, MacWhisper) — user may still need some
- App data for apps still in use (Spotify offline, streaming caches)
- Large datasets in `~/.cache/huggingface/hub/datasets--*`

### Category: Caution (user data, could be important)

- Documents, Downloads, Movies
- Application databases
- Project directories

### iCloud optimization opportunities

If optimize storage is enabled, mention large local files in iCloud-synced directories that could be evicted via Finder to free space without losing data.

## Phase 3: Clean

Wait for user to choose what to clean. Then:

1. Show the user the exact commands that will be run and the estimated space to be reclaimed
2. Wait for final confirmation before executing
3. Execute cleanup commands for selected items
4. After cleanup, re-check space with `diskutil info /`
5. Report before/after comparison: how much free space was gained

### Quick clean commands reference

```bash
# Package manager caches
pip3 cache purge
npm cache clean --force
rm -rf ~/Library/Caches/Homebrew/
conda clean --all -y

# HuggingFace transfer cache
rm -rf ~/.cache/huggingface/xet/

# Ollama model removal
ollama rm <model-name>

# Browser caches (safe, auto-rebuild)
rm -rf ~/Library/Caches/Google/
rm -rf ~/Library/Caches/Firefox/
rm -rf ~/Library/Caches/Microsoft\ Edge/
rm -rf ~/Library/Caches/BraveSoftware/
```

Note: Google Chrome cache may require Chrome to be quit first due to file locks.

## Important rules

- Never delete anything without user confirmation — always confirm each category individually, even if user says "delete all"
- Always report the before/after free space delta using `diskutil info /`
- When unsure if something is safe to delete, research it first (check if the app is still installed, if the command/binary exists, etc.)
- For items you're not confident about, say so — don't guess
- Do not use `brctl evict` — it does not exist. iCloud eviction is Finder-only.
- Do not use `df -h` for free space reporting — it can be misleading on APFS. Use `diskutil info /` instead.

## Summary Mode

When invoked in summary mode (from full checkup), run only Phase 1 steps 1.1 (check available space) and 1.2 (scan large directories). Report total free space and top 5 largest directories. Skip drill-downs (1.3), orphan detection (1.4), iCloud check (1.5), and Phases 2-3.
