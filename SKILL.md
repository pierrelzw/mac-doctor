---
name: mac-doctor
description: >
  macOS system doctor — system info, disk cleanup, and maintenance.
  Use when user mentions "系统信息", "磁盘清理", "清理磁盘", "系统维护",
  "disk full", "free up space", "clean disk", "system info", "system status",
  "mac doctor", "体检", "checkup", "DNS flush", "Spotlight rebuild".
user-invocable: true
allowed-tools: Bash, Read, Glob
---

# mac-doctor

macOS system doctor: diagnose, clean, and maintain your Mac.

## Language Rule

Detect the language of $ARGUMENTS. If $ARGUMENTS is empty, detect from the user's message that triggered this skill invocation.

- Chinese input (any variant) → respond entirely in Chinese
- All other languages or ambiguous → respond in English

Apply this language choice throughout the entire session.

## Error Handling

Global rules for all modules:

- If a command fails or returns empty output, display "N/A" for that item and briefly note the reason. Never silently omit a row.
- If an optional tool is not installed (e.g., ollama, conda, pip3), skip that item and note "not installed". Do not treat this as an error.
- If a command is running unusually long (e.g., `du` on a nearly-full disk), inform the user and consider reducing scan scope.

## Routing

Parse $ARGUMENTS and route to the matching module:

| Keywords | Module |
|----------|--------|
| disk, 磁盘, clean, storage, 存储, 空间, space, cache, 缓存 | disk-clean |
| info, 信息, status, 状态, overview, hardware, 硬件, specs, 配置 | system-info |
| maintain, 维护, dns, spotlight, repair, fix, 修复, diagnose, 诊断, rebuild, 重建 | maintenance |
| all, 全面, checkup, 体检, health, 健康, scan, 扫描, 检查, check, OR empty $ARGUMENTS | all modules (summary mode) |

Match is case-insensitive and partial (e.g., "磁盘满了" contains "磁盘" → disk-clean).

**Precedence**: if keywords match multiple modules, prefer the longest keyword match. If still tied, prefer this order: maintenance > disk-clean > system-info (most specific action wins).

**Fallback**: if $ARGUMENTS matches no keyword, list the three available modules with a brief description and ask the user to clarify.

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
