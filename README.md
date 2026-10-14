# Windows 中文环境踩坑记

> 33 条在 **Windows 11 + git-bash（MSYS）+ 中文环境**下做终端自动化、脚本编排与 AI Agent 开发时**实际踩过、记录下来、并且修好**的坑。
>
> 每条都带症状、根因、修法，多数附实测日期。没有"理论上可能"的条目——没踩过的不写。

## English

Hands-on notes from running terminal automation and AI-agent workflows on **Windows 11 with git-bash (MSYS)** in a **Chinese-language environment**. 33 entries, each with symptom → root cause → fix, most tagged with the date it was hit and verified.

The recurring themes are the ones that waste the most time:

- **MSYS path mangling** — `/c/x` vs `C:/x`, `//flag` mangling, `cmd //c`, PATH reassembly, `/tmp` pointing at a different drive than native tools expect
- **Encoding traps** — Chinese paths and filenames breaking toolchains, BOM-induced "Binary file" false positives, UTF-16 misdetection
- **Zero-window discipline** — why launching `powershell.exe` from a terminal pops a visible console and can take down the parent app with a stray Ctrl+C, and what to do instead
- **PowerShell ↔ bash quoting** — three-layer escaping (`$_`, `\\`, regex), execution policy blocking `npm`/`npx`, `ConvertTo-Json` round-trips
- **Self-inflicted damage** — regex-based process killing that kills the caller, mass-edit operations that mangle neighboring lines

Text is in Chinese (that is the target audience). Code, commands and error strings are reproduced verbatim.

## 目录

### 终端与路径（git-bash / MSYS）
- [1. terminal 命令含中文路径 → lifecycle_guard 报错](docs/windows-cn-pitfalls.md#1-terminal-命令含中文路径--lifecycle_guard-报错)
- [2. MSYS 路径转换坑](docs/windows-cn-pitfalls.md#2-msys-路径转换坑)
- [21. git-bash 里 `cmd //c "..."` 被 MSYS 破坏 → cmd 进交互模式](docs/windows-cn-pitfalls.md#21-git-bash-里-cmd-c--被-msys-破坏--cmd-进交互模式)
- [25. bash 双引号包 PowerShell：`$_` 被 bash 先展开](docs/windows-cn-pitfalls.md#25-bash-双引号包-powershell_-被-bash-先展开)
- [26. 拼 PATH 调 npm 会被 MSYS 拆坏；计划任务脚本必须全硬编码绝对路径](docs/windows-cn-pitfalls.md#26-拼-path-调-npm-会被-msys-拆坏计划任务脚本必须全硬编码绝对路径)
- [30. `curl -o /dev/null` 在 MSYS 下写不进去（exit 23）](docs/windows-cn-pitfalls.md#30-curl--o-devnul-在-msys-下写不进去exit-23别用它丢弃输出)

### 编码与中文处理
- [3. read_file 报 "Binary file" 但文件其实是文本](docs/windows-cn-pitfalls.md#3-read_file-报-binary-file-但文件其实是文本)
- [4. 中文网页抓取与解析](docs/windows-cn-pitfalls.md#4-中文网页抓取与解析)
- [5. python-docx 生成中文文档](docs/windows-cn-pitfalls.md#5-python-docx-生成中文文档)
- [8. 分类/标题行正则遇换行匹配失败](docs/windows-cn-pitfalls.md#8-分类标题行正则遇换行匹配失败2026-08-05-实测)

### Python、依赖与工具行为
- [6. 安装缺失 Python 库](docs/windows-cn-pitfalls.md#6-安装缺失-python-库)
- [7. 老格式 .xls 与 .xlsx 区分](docs/windows-cn-pitfalls.md#7-老格式-xls-与-xlsx-区分2026-08-05-实测)
- [9. write_file 对 .js 的 lint 检查在 MSYS 下误报](docs/windows-cn-pitfalls.md#9-write_file-对-js-的-lint-检查在-msys-下误报)
- [10. patch 工具改含 Windows 路径的 Python 源码会损坏 `\r` 转义](docs/windows-cn-pitfalls.md#10-patch-工具改含-windows-路径的-python-源码会损坏-r-转义2026-08-05-实测)
- [11. execute_code 内联代码含中文路径 → 安全拦截超时](docs/windows-cn-pitfalls.md#11-execute_code-内联代码含中文路径--安全拦截超时2026-08-05-实测)
- [14. HF 模型下载慢/中断](docs/windows-cn-pitfalls.md#14-hf-模型下载慢中断2026-08-05-实测whisper-模型)

### 批量改写的连环误伤
- [12. 列表/编号重排的连环误伤](docs/windows-cn-pitfalls.md#12-列表编号重排的连环误伤2026-08-05-实测)
- [13. 删代码块时误删关键语句](docs/windows-cn-pitfalls.md#13-删代码块时误删关键语句2026-08-05-实测)
- [23. PowerShell ConvertTo-Json 的 `\\` 转义 + bash grep 三层打架](docs/windows-cn-pitfalls.md#23-powershell-convertto-json-的--转义--bash-grep-三层打架2026-08-23-实测)

### PowerShell 与计划任务
- [18. wmic 已移除 → 用 PowerShell Get-CimInstance 采硬件](docs/windows-cn-pitfalls.md#18-wmic-已移除--用-powershell-get-ciminstance-采硬件2026-08-07-实测)
- [19. PowerShell 执行策略禁跑 npm/npx](docs/windows-cn-pitfalls.md#19-powershell-执行策略禁跑-npmnpx2026-08-22-实测)
- [29. 在 Windows 上跑脚本/命令行工具必须"零窗口"](docs/windows-cn-pitfalls.md#29-在-windows-上跑脚本命令行工具必须零窗口2026-09-13-实测攒了-4-个控制台窗口)

### 进程与窗口
- [24. 按 CommandLine 正则杀进程会连自己一起杀；`schtasks /end` 不杀孙进程](docs/windows-cn-pitfalls.md#24-按-commandline-正则杀进程会连自己一起杀schtasks-end-不杀孙进程)
- [31. 用脚本/计划任务重启 GUI 客户端要"像双击一样"启动](docs/windows-cn-pitfalls.md#31-用脚本计划任务重启-gui-客户端要像双击一样启动2026-09-13-实测关窗口把客户端带走了)

### 脚本编写
- [22. .bat 脚本里括号嵌套崩溃](docs/windows-cn-pitfalls.md#22-bat-脚本里-if-errorlevel-8-echo-failxxx--括号嵌套崩溃2026-08-23-实测)
- [27. heredoc 写脚本会被吞反斜杠 → 一律用 write_file](docs/windows-cn-pitfalls.md#27-heredoc-写脚本会被吞反斜杠--一律用-write_file2026-09-15-实测)

### 网络与代理
- [16. git-bash curl 悄悄走系统代理 + Clash TUN 假 IP](docs/windows-cn-pitfalls.md#16-git-bash-curl-悄悄走系统代理--clash-tun-假-ip2026-08-05-实测steam-案例)
- [17. QQ 文件附件收不到（SSRF 白名单 + 网关重启）](docs/windows-cn-pitfalls.md#17-qq-文件附件收不到ssrf-白名单--网关重启2026-08-07-实测)

### 硬件与设备
- [28. USB 设备"认不到"的取证顺序](docs/windows-cn-pitfalls.md#28-usb-设备认不到的取证顺序手机硬盘手柄通用2026-09-15-实测)

### 其他
- [15. 电话/方言音频转写模型选择](docs/windows-cn-pitfalls.md#15-电话方言音频转写模型选择2026-08-05-实测)
- [20. npx 后台首次安装大 monorepo 的进度监控](docs/windows-cn-pitfalls.md#20-npx-后台首次安装大-monorepo-的进度监控2026-08-22-实测)
- [32. 按内存自适应的默认值在 Windows 上会退化成无上限](docs/windows-cn-pitfalls.md#32-按内存自适应的默认值在-windows-上会退化成无上限)
- [33. 统计「有多少文件 / 多大体积」前先定口径](docs/windows-cn-pitfalls.md#33-统计有多少文件--多大体积前先定口径两个会报错数的坑)

## 扩展阅读

- [`docs/clash-proxy-and-steam-access.md`](docs/clash-proxy-and-steam-access.md) —— Clash/mihomo TUN 模式和 git-bash 系统代理叠加时的排障路径（curl 直连、fake-ip 段识别、TUN 伪造 TCP 握手导致端口扫描结论作废）
- [`docs/uv-venv-msys-path-and-pythonpath.md`](docs/uv-venv-msys-path-and-pythonpath.md) —— `uv venv` 在 git-bash 下把 venv 建到 `C:\c\Users\...`，以及 `PYTHONPATH` 污染独立 venv

## 关于工具名

文中出现的 `terminal`、`read_file`、`write_file`、`patch`、`execute_code`、`web_extract` 是 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的工具名——这些坑是在用它的过程中踩出来的。**绝大部分与具体 agent 无关**：换成任何在 Windows 上跑命令的框架，路径转换、编码、零窗口、代理这些问题一模一样。

## 收录标准

只收"踩过 → 定位到根因 → 验证修法"的条目。感受性的、一次性的、没法复现的都不收。条目的实测日期随原始记录保留。

## License

MIT — 见 [LICENSE](LICENSE)。
