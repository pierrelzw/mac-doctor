# mac-doctor — Claude Code Skill 设计计划

## Context

现有 `clean-disk` skill 只做磁盘清理，且输出固定为英文。目标是创建一个更通用的 macOS 系统维护 skill：
- 扩展为三个能力模块：系统信息总览、磁盘清理、macOS 维护
- 根据用户输入语言自动切换输出语言（面向开源）
- 支持 `$ARGUMENTS` 参数路由到不同模块
- 后续可扩充更多模块

## 文件结构

```
mac-doctor/
├── SKILL.md              # 入口：语言规则 + 参数路由
├── modules/
│   ├── system-info.md    # 模块1: 系统信息总览
│   ├── disk-clean.md     # 模块2: 磁盘清理
│   └── maintenance.md    # 模块3: macOS 维护
├── PLAN.md               # 本文件
└── README.md             # 开源说明（可选）
```

## SKILL.md 设计

### Frontmatter

```yaml
name: mac-doctor
description: >
  macOS system doctor — system info, disk cleanup, and maintenance.
  Use when user mentions "系统信息", "磁盘清理", "清理磁盘", "系统维护",
  "disk full", "free up space", "clean disk", "system info", "system status",
  "mac doctor", "体检", "checkup", "DNS flush", "Spotlight rebuild".
```

### 语言自适应规则

在 SKILL.md 顶部设定规则：

```
Detect the language of $ARGUMENTS (or the user's latest message if $ARGUMENTS is empty).
Respond entirely in that language throughout the session.
- Chinese input (any variant) → Chinese output
- All other languages → English output
```

不写两套模板，利用 Claude 的多语言能力，只需一条明确指令。

### 参数路由逻辑

```
Parse $ARGUMENTS and route to module(s):

Keywords → Module:
  disk, 磁盘, clean, storage, 存储, 空间  → disk-clean.md
  info, 系统, status, 信息, overview       → system-info.md
  maintain, 维护, fix, dns, spotlight, repair → maintenance.md
  all, 全面, checkup, 体检, (empty)        → run all three modules (summary mode)

Read the matched module file and execute its instructions.
When running all modules, use summary mode (skip detailed drill-downs, only top-level checks).
```

## 模块设计

### 模块1: system-info.md — 系统信息总览

快速收集并展示系统状态：

| 检查项 | 命令 |
|--------|------|
| macOS 版本 | `sw_vers` |
| 硬件型号 | `sysctl -n hw.model` + `system_profiler SPHardwareDataType` (chip info) |
| CPU 使用率 | `top -l 1 -n 0 \| grep "CPU usage"` |
| 内存使用 | `vm_stat` + `sysctl -n hw.memsize` |
| 存储概览 | `diskutil info / \| grep "Container"` |
| 电池健康 | `system_profiler SPPowerDataType \| grep -E "Cycle Count\|Condition\|Maximum Capacity"` |
| 运行时间 | `uptime` |
| 网络连接 | `networksetup -getinfo Wi-Fi` |

输出格式：一张清晰的表格，一目了然。

### 模块2: disk-clean.md — 磁盘清理

从现有 clean-disk skill 迁移，保留全部逻辑：

1. **扫描阶段**：检查可用空间 → 扫描大目录 → 钻取已知空间杀手 → 检测孤儿应用数据 → iCloud 状态
2. **报告阶段**：按安全级别分类（安全删除 / 需确认 / 谨慎操作）
3. **清理阶段**：等用户选择 → 执行清理 → 报告前后对比

规则不变：
- 未经用户确认不删除任何东西
- 用 `diskutil info /` 而非 `df -h`
- 不确定的标注为不确定

### 模块3: maintenance.md — macOS 维护

常见维护操作，分为「诊断」和「修复」两部分：

**诊断检查：**
| 检查项 | 命令 |
|--------|------|
| 磁盘健康 | `diskutil verifyVolume /` |
| 系统完整性保护 | `csrutil status` |
| 启动项 | `osascript -e 'tell application "System Events" to get the name of every login item'` |
| 最近崩溃日志 | `ls -lt ~/Library/Logs/DiagnosticReports/ \| head -5` |
| 系统日志错误 | `log show --last 1h --predicate 'eventType == logEvent AND logType == error' --style compact \| tail -20` |

**修复操作（需用户确认）：**
| 操作 | 命令 | 风险级别 |
|------|------|----------|
| 刷新 DNS 缓存 | `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder` | 低 |
| 重建 Spotlight 索引 | `sudo mdutil -E /` | 低（但耗时） |
| 重建 Launch Services | `/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -kill -r -domain local -domain system -domain user` | 低 |
| 清除字体缓存 | `sudo atsutil databases -remove` | 低 |
| 修复磁盘权限 | `diskutil repairPermissions /`（仅 macOS < 10.15 有效） | 低 |

规则：
- 所有 `sudo` 操作必须先告知用户并获得确认
- 修复操作逐个执行，每个执行前确认
- 标注 macOS 版本兼容性（如修复权限仅适用于旧系统）

## 语言适配实现细节

不在模块文件中写双语内容。模块用英文编写（作为结构化指令），SKILL.md 中的语言规则确保 Claude 用用户语言输出。这样：
- 模块文件保持简洁，不膨胀
- 利用 Claude 天然的翻译能力
- 新增语言零成本

## 安装方式

```bash
# 克隆到本地
git clone <repo-url> ~/codes/mac-doctor

# 创建 symlink 到 Claude skills 目录
ln -s ~/codes/mac-doctor ~/.claude/skills/mac-doctor
```

## 验证方式

1. **语言测试**：`/mac-doctor 看看系统状态` → 应输出中文；`/mac-doctor check system` → 应输出英文
2. **路由测试**：`/mac-doctor disk` → 只跑磁盘模块；`/mac-doctor 维护` → 只跑维护模块
3. **无参数测试**：`/mac-doctor` → 跑全面体检精简版
4. **磁盘清理流程**：`/mac-doctor 磁盘清理` → 完整扫描 → 报告 → 等待用户选择 → 清理 → 对比
5. **维护操作确认**：`/mac-doctor maintenance` → 诊断 → 提供修复选项 → 逐个确认执行
