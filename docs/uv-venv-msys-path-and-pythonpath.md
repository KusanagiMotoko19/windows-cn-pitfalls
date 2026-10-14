# uv venv 在 git-bash 下建错目录（C:\c\...）——2026-08-23 实测

## 症状
- git-bash 里执行 `uv venv ~/xxx-venv` 或 `uv venv /c/Users/xxx/xxx-venv`，venv 被建到 **`C:\c\Users\<USER>\xxx-venv`**（盘符根下多出 `c\` 目录）
- uv 输出 "Creating virtual environment at: xxx-venv"（路径被相对化显示，看不出错）
- `ls ~/xxx-venv` 报 No such file or directory
- 后续 `--python ~/xxx-venv/Scripts/python.exe` 自然全失败

## 根因
uv 是 Windows 原生 exe。MSYS/git-bash 把 `/c/Users/...`、`~/...` 这类路径在传给原生程序时做转换；但当参数以 `/c/` 开头时转换结果变成相对路径（`c/Users/...`），uv 把它拼到当前盘符根 `C:\` 下 → `C:\c\Users\...`。

## 解法（venv 路径必须用 Windows 原生形式）
```bash
uv venv "$USERPROFILE\xxx-venv" --python 3.11
# $USERPROFILE 在 bash 双引号里展开为 C:\Users\<USER>，MSYS 不动带盘符的路径
uv pip install --python "$USERPROFILE\xxx-venv\Scripts\python.exe" <pkg>
```
或写死正斜杠 Windows 路径：`uv venv "C:\Users\<USER>/xxx-venv"`。

## 适用范围
任何 Windows 原生 CLI 在 git-bash 里接收路径参数都有同类问题（uv、npx/npm 等）。规则：**路径参数优先用 `$USERPROFILE\...` 或 `C:/...`，不要用 `~/...` 或 `/c/...`**。

## 验证
装完立即 `ls ~/xxx-venv/Scripts/python.exe`（或 `"$USERPROFILE\xxx-venv\Scripts\python.exe" -c "print(1)"`），不要信 uv 的输出。

## 连带坑：Hermes 系统 PYTHONPATH 污染独立 venv
本机 shell 的 `PYTHONPATH` 指向 Hermes 自己的 venv（`C:\Users\<USER>\AppData\Local\hermes\hermes-agent;C:\...\hermes-agent\venv\Lib\site-packages`）。任何独立 venv 的 Python 进程 import 时都会先命中 Hermes 的包 → 版本错乱（实测：omni-venv 里 import tokenizers 加载到 Hermes 的 0.23.1，尽管 venv 里装的是 0.23.0rc0）。
**解法：运行/安装一律 `python -E`（忽略所有 PYTHON* 环境变量）**，比 `PYTHONPATH=` 前缀更彻底；启动脚本里写死 `exec venv/Scripts/python.exe -E app.py`。
