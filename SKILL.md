---
name: mac-doctor
description: >
  macOS system doctor — system info, disk cleanup, and maintenance.
  Use when user mentions "系统信息", "磁盘清理", "清理磁盘", "系统维护",
  "disk full", "free up space", "clean disk", "system info", "system status",
  "mac doctor", "体检", "checkup", "DNS flush", "Spotlight rebuild".
user-invocable: true
---

# mac-doctor

macOS system doctor: diagnose, clean, and maintain your Mac.

## Language Rule

Detect the language of $ARGUMENTS (or the user's latest message if $ARGUMENTS is empty).

- Chinese input (any variant) → respond entirely in Chinese
- All other languages → respond in English

Apply this language choice throughout the entire session.

## Routing

Parse $ARGUMENTS and route to the matching module:

| Keywords | Module |
|----------|--------|
| disk, 磁盘, clean, storage, 存储, 空间 | disk-clean |
| info, 系统, status, 信息, overview | system-info |
| maintain, 维护, fix, dns, spotlight, repair | maintenance |
| all, 全面, checkup, 体检, OR empty $ARGUMENTS | all modules (summary mode) |

Match is case-insensitive and partial (e.g., "磁盘满了" contains "磁盘" → disk-clean).

### Single module

Read the matched module file and follow its full instructions:

```bash
cat ${CLAUDE_SKILL_DIR}/modules/<module-name>.md
```

### Full checkup (all modules, or empty $ARGUMENTS)

Run all three modules in **summary mode** (each module defines what summary mode means):

1. Read and execute `${CLAUDE_SKILL_DIR}/modules/system-info.md` (summary mode)
2. Read and execute `${CLAUDE_SKILL_DIR}/modules/disk-clean.md` (summary mode)
3. Read and execute `${CLAUDE_SKILL_DIR}/modules/maintenance.md` (summary mode)

Present a unified report with three sections. End with an overall health assessment.
