# [CLAUDE.md](http://CLAUDE.md)

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

**End-user** (via plugin marketplace):

```bash
claude plugin marketplace add pierrelzw/zhiwei-skills
claude plugin install mac-doctor@pierrelzw --scope user
```

**Developer** (local symlink for active development):

```bash
git clone https://github.com/pierrelzw/mac-doctor ~/codes/mac-doctor
ln -s ~/codes/mac-doctor ~/.claude/skills/mac-doctor
```

> Do not use both install methods simultaneously — remove the symlink before plugin install, or vice versa.

## Publishing

Plugin metadata lives in `.claude-plugin/plugin.json`. This is the canonical version/metadata file.

**Version bump workflow**:

1. Update `version` in `.claude-plugin/plugin.json`
2. Sync the version in `~/codes/zhiwei-skills/.claude-plugin/marketplace.json` (mac-doctor entry)
3. Validate: `claude plugin validate .claude-plugin/plugin.json`
4. Commit and push both repos

## Key Rules

- **Never delete files without explicit user confirmation** — disk-clean module must wait for user selection before executing any cleanup.
- **All ****`sudo`**** operations require user confirmation before execution** — maintenance repairs are executed one at a time with individual confirmation.
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

## Evaluation

`evals/` 目录包含自动评测框架，用于 skill 修改前后的效果对比。

- `evals/criteria.md` — 7 个评分维度 + 8 个测试场景的评测标准
- `evals/results/` — 历史评测结果（按 `<date>-<git-hash>.md` 命名）

**使用方法**：修改 skill 后，告诉 Claude "评测一下"。Claude 会：

1. 读取所有 skill 文件，对照 `criteria.md` 逐项打分
2. 保存结果到 `evals/results/`
3. 与上次评测结果自动对比，报告改善/退步

