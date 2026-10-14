# Clash 代理排查 + Steam 数据访问（2026-08-05 实测）

场景：国内 Windows 机器（Clash Party / mihomo TUN 模式）上请求访问被墙站点（Steam 等）。
本文档 = 完整诊断命令 + Steam 本机数据读取 + ID 换算 + 在线拉取方案（2026-08-05 全部实测验证）。

## 一、症状与快速判断

| 症状 | 含义 |
|---|---|
| web_extract 报 "Blocked: URL targets a private or internal network address" | 域名被 Clash fake-ip DNS 解析成 198.18.x.x，被安全策略当内网地址拦（不是真的内网！） |
| 浏览器 ERR_CONNECTION_CLOSED | 浏览器走系统代理 → 代理端口 TLS 转发坏 |
| curl 全部 000（含 google） | curl 默认走了坏的**系统代理** 127.0.0.1:7890，不是断网 |
| `--noproxy "*"` 后 google 200 但 Steam 域名 000 | mihomo 规则把 Steam 路由到策略组，该组被切到 DIRECT → 直连被墙 |

## 二、诊断命令（git-bash）

```bash
# 1. 系统代理状态（curl 会读这个，即使 env 没 proxy）
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable

# 2. 关键：绕过系统代理走 TUN 测真实网络
curl -sI --noproxy "*" --max-time 8 https://www.google.com -o /dev/null -w "google: %{http_code}\n"
curl -sI --noproxy "*" --max-time 8 https://steamcommunity.com -o /dev/null -w "steam: %{http_code}\n"

# 3. 确认 TUN fake-ip（解析出 198.18.x.x = TUN 接管）
nslookup steamcommunity.com

# 4. 代理进程与配置
tasklist | grep -iE "clash|mihomo"
powershell -NoProfile -c "(Get-CimInstance Win32_Process -Filter \"name='mihomo.exe'\").CommandLine"
# → 常见：mihomo.exe -d %APPDATA%\mihomo-party\work -ext-ctl-pipe \\.\pipe\...
# 生效配置 = %APPDATA%\mihomo-party\work\config.yaml；订阅 profile 在 profiles/*.yaml

# 5. 查域名路由规则（在 work/config.yaml 里）
grep -nE "RULE-SET,Steam" %APPDATA%/mihomo-party/work/config.yaml
grep -n "游戏平台" %APPDATA%/mihomo-party/work/config.yaml
# 🎮 游戏平台 组为 select 型、第一选项 DIRECT——若 GUI 里被切到 DIRECT，Steam 全系走直连被墙
```

结论路径：全部 000 → `--noproxy "*"` 后 google 200 → 说明系统代理端口坏（CONNECT 通但 TLS 握手失败），真实网络好；Steam 域名仍 000 → 规则直连被墙。**在 Clash Party GUI 把「🎮 游戏平台」策略组切回代理节点**（外部控制器是命名管道，无 REST 可远程切）。不要硬改 work/config.yaml 或 override.yaml（后者 GUI 管理，格式 items: []）。

## 三、Steam 本机数据（已实测可用）

```bash
# 安装路径（注册表）
reg query "HKCU\Software\Valve\Steam" /v SteamPath          # → g:/steam
reg query "HKLM\SOFTWARE\WOW6432Node\Valve\Steam" /v InstallPath

# 已安装游戏列表（appmanifest_*.acf 含 appid / "name" / "SizeOnDisk"）
for f in /g/Steam/steamapps/appmanifest_*.acf; do
  appid=$(basename "$f" .acf | sed 's/appmanifest_//')
  name=$(grep -m1 '"name"' "$f" | sed 's/.*"name"[[:space:]]*"\(.*\)"/\1/')
  echo "$appid | $name"
done

# 多库路径
grep -o '"path"[^,]*' /g/Steam/steamapps/libraryfolders.vdf
```

- `userdata/<数字>` 目录 = Steam 账号（每账号一个），数字是 SteamID3 的 account id 段
- **SteamID64 = 76561197960265728 + accountid**（例：accountid 38XXXXXXX → **7656119XXXXXXXXXX**）
- ⚠️ **SteamID 换算必须用脚本算 + loginusers.vdf 交叉验证，不要手算**（2026-08-05 实测手算错 234000，把 7656119XXXXXXXXXX 算成 7656119ZZZZZZZZZZ，后者是另一个账号 <ALT_PERSONA_SHORT>，导致抓错号）。交叉验证：
  ```bash
  grep -A4 "<accountid>" /g/Steam/config/loginusers.vdf   # SteamID64 → AccountName/PersonaName
  # 本机主账号：7656119XXXXXXXXXX = <MAIN_PERSONA> / "<MAIN_PERSONA_SHORT>"
  # 另一账号：7656119YYYYYYYYYY = <ALT_PERSONA> / <ALT_LABEL>
  ```
- 个人资料页：`https://steamcommunity.com/profiles/<steamid64>/`（资料公开时 200，并跳转 `/id/<customURL>/`）
- 注意：appmanifest 只有**已安装**的游戏，≠ 完整拥有库

## 四、在线拉取完整库（2026-08-05 已验证）

**结论：网页接口全部走不通（Steam 反爬风控），API key 是唯一正路。**

- ❌ 网页/XML 接口：`https://steamcommunity.com/profiles/<steamid64>/games/?tab=all&xml=1` —— 即使资料「游戏详情」公开、域名能走代理，**机房/代理 IP + 无登录 cookie 一律 302 跳 `/login/`**（Steam 反爬）。主页 `?xml=1`（返回 steamID/customURL/memberSince/location）不受影响，可用它确认资料公开性与账号身份
- ✅ **API（唯一正路）**：Steam Web API key（steamcommunity.com/dev/apikey 免费注册，域名栏随便填如 localhost）+ `IPlayerService/GetOwnedGames/v1/?key=K&steamid=<steamid64>&include_appinfo=1&include_played_free_games=1&format=json` → 全量库 + 各游戏游玩时长。`api.steampowered.com` 服务器间 API **不受网页风控**，走代理（策略组切回后）实测连通（`ISteamWebAPIUtil/GetServerInfo/v1/` 返回 200）
- 注册页在 steamcommunity（需登录）：外部浏览器打不开时用 `steam://openurl/https://steamcommunity.com/dev/apikey`（Win+R 运行）→ Steam 客户端内置浏览器（已登录态）打开
- 区分多账号：userdata 下多个目录时，用 loginusers.vdf 的 PersonaName 与实际身份核对再确认主账号

**已验证的死路（别再试）**：
- Steam 客户端登录态 cookie：`%LOCALAPPDATA%\Steam\htmlcache\Default\Network\Cookies`（Chromium 格式）被 steamwebhelper **进程独占锁**，直接读/cp/robocopy /B 全失败
- CDP 调试：steamwebhelper.exe 启动参数**无 `--remote-debugging-port`**（默认关闭），无法连接调试端口读已登录页面
- 本机 `config/htmlcache` 不存在（新版 Steam 的 web 缓存只在 `%LOCALAPPDATA%\Steam\htmlcache`）
