# mac-doctor

macOS system doctor for Claude Code — system info, disk cleanup, and maintenance.

## Install

```bash
claude plugin marketplace add pierrelzw/zhiwei-skills && claude plugin install mac-doctor@pierrelzw --scope user
```

## Usage

```
/mac-doctor              # Run all modules in summary mode
/mac-doctor system       # System info only (CPU, memory, storage, battery, network)
/mac-doctor disk         # Disk cleanup (scan → report → user-confirmed cleanup)
/mac-doctor maintenance  # Diagnostics + repair (DNS flush, Spotlight rebuild, etc.)
```

Supports bilingual input:

```
/mac-doctor 系统信息      # 系统信息模块（中文输出）
/mac-doctor 磁盘清理      # 磁盘清理模块
/mac-doctor 维护          # 系统维护模块
```

## Modules

| Module | Description |
|--------|-------------|
| system-info | CPU, memory, storage, battery, network overview |
| disk-clean | Scan for large caches, AI models, app residuals; user-confirmed cleanup |
| maintenance | DNS flush, Spotlight rebuild, permission repair, and more |

## License

MIT
