# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

mac-doctor is a **Claude Code skill** for macOS system maintenance. It is not a traditional application — it consists of Markdown instruction files that Claude Code reads and executes. There is no build step, compilation, or package manager.

## Architecture

```
SKILL.md                  # Entry point: language detection + argument routing
modules/
  system-info.md          # System overview (CPU, memory, storage, battery, network)
  disk-clean.md           # Disk cleanup (scan → report → user-confirmed cleanup)
  maintenance.md          # Diagnostics + repair operations (DNS flush, Spotlight rebuild, etc.)
```

**Routing**: `SKILL.md` parses `$ARGUMENTS` keywords to route to the correct module. Empty arguments run all modules in summary mode.

**Language**: Modules are written in English as structural instructions. `SKILL.md` contains a single language-detection rule — Claude translates output to match the user's input language. No dual-language templates.

## Installation

```bash
git clone <repo-url> ~/codes/mac-doctor
ln -s ~/codes/mac-doctor ~/.claude/skills/mac-doctor
```

## Key Rules

- **Never delete files without explicit user confirmation** — disk-clean module must wait for user selection before executing any cleanup.
- **All `sudo` operations require user confirmation before execution** — maintenance repairs are executed one at a time with individual confirmation.
- Use `diskutil info /` instead of `df -h` for storage info.
- Mark macOS version compatibility where relevant (e.g., `diskutil repairPermissions` only works on macOS <= 10.11).
- When uncertain about a file/directory's purpose, label it as uncertain rather than guessing.

## Testing

Manual verification only — no automated test framework:

1. Language: `/mac-doctor 看看系统状态` → Chinese output; `/mac-doctor check system` → English output
2. Routing: `/mac-doctor disk` → disk module only; `/mac-doctor 维护` → maintenance module only
3. No args: `/mac-doctor` → all modules in summary mode
4. Disk cleanup: full scan → categorized report → user selection → cleanup → before/after comparison
5. Maintenance: diagnostics → repair options → per-operation confirmation → execution
