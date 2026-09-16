# 后台服务与守护进程清单

> 两台机器的 launchd 服务、brew 服务、cron 任务等后台进程记录；实际加载状态以 launchctl 现场查看为准。

---

## 一、本机（cw / 192.168.2.8）

### 1.1 用户级 launchd 服务（~/Library/LaunchAgents/）

| 服务 Label | plist 文件 | 用途 | 状态 |
|---|---|---|---|
| `com.opentoken.daemon` | `com.opentoken.daemon.plist` | TokenRank 用量上报，每 30 分钟（每小时第 2/32 分） | 已加载 |
| `io.github.clash-verge-rev.clash-verge-rev` | `io.github.clash-verge-rev.clash-verge-rev.plist` | Clash Verge Rev 代理工具 | 已加载 |

**OpenToken 服务详情：**
- 程序：`$HOME/.local/bin/opentoken daemon --once`
- 调度：`StartCalendarInterval` 每小时第 2 分、第 32 分
- 日志：`$HOME/.opentoken/daemon.log`
- 配置：`$HOME/.opentoken/config.json`

### 1.2 Homebrew 服务

以下 brew 服务已安装但当前状态为 `none`（未启动）：

| 服务 | 用途 |
|---|---|
| `bitcoin` | Bitcoin 节点 |
| `emacs` | Emacs 守护进程 |
| `mysql` | MySQL 数据库 |
| `nginx` | Nginx Web 服务器 |
| `postgresql@14` | PostgreSQL 14 数据库 |
| `unbound` | Unbound DNS 解析器 |

管理命令：
```bash
brew services list                    # 查看所有
brew services start <name>           # 启动并设为开机自启
brew services stop <name>            # 停止
brew services restart <name>         # 重启
```

### 1.3 cron 任务

| 调度 | 命令 | 用途 |
|---|---|---|
| `25 0 * * *`（每天 00:25） | `$HOME/.acme.sh/acme.sh --cron --home "$HOME/.acme.sh"` | acme.sh SSL 证书自动续期 |

查看：`crontab -l`
编辑：`crontab -e`

---

## 二、远程机（wj / 192.168.2.9，黑苹果）

### 2.1 用户级 launchd 服务（~/Library/LaunchAgents/）

| 服务 Label | plist 文件 | 用途 | 状态 |
|---|---|---|---|
| `com.hp.productresearch` | `com.hp.productresearch.plist` | HP 打印机相关（惠普产品研究） | 已加载 |
| `com.user.cloudflared-power` | `com.user.cloudflared-power.plist` | Cloudflare Tunnel，暴露 power-webhook 服务到公网 | 已加载，KeepAlive |
| `com.user.powerwebhook` | `com.user.powerwebhook.plist` | Python 关机 Webhook 服务，接收远程关机指令 | 已加载，KeepAlive |

**cloudflared-power 详情：**
- 程序：`/usr/local/bin/cloudflared tunnel run power-webhook`
- 日志：`$HOME/Library/Logs/cloudflared-power.log` 和 `.err.log`
- 作用：将本机的 powerwebhook 服务通过 Cloudflare Tunnel 暴露到公网域名

**powerwebhook 详情：**
- 程序：`/usr/bin/python3 $HOME/bin/shutdown-webhook.py`
- 环境变量：`SHUTDOWN_TOKEN`（认证令牌，敏感，见 credentials.md）
- 日志：`$HOME/Library/Logs/powerwebhook.log` 和 `.err.log`
- 作用：监听 HTTP 请求，验证 token 后执行关机操作（配合 cloudflared tunnel 实现远程关机）

### 2.2 Homebrew

远程机**已装** Homebrew（/usr/local，约 170 formulae + 6 casks）。非交互 SSH 的 PATH 不含 /usr/local/bin，远程查 brew 服务用绝对路径：
```bash
ssh wj '/usr/local/bin/brew services list'
```

### 2.3 cron 任务

无。

---

## 三、服务管理通用 SOP

### 查看服务状态
```bash
# launchd 用户服务
launchctl list | grep <keyword>
ls -la ~/Library/LaunchAgents/

# 查看服务详情
launchctl list <label>
cat ~/Library/LaunchAgents/<label>.plist
```

### 启动/停止/重启 launchd 服务
```bash
# 加载（注册并启动）
launchctl load ~/Library/LaunchAgents/<label>.plist

# 卸载（停止并注销）
launchctl unload ~/Library/LaunchAgents/<label>.plist

# 重启（先 unload 再 load）
launchctl unload ~/Library/LaunchAgents/<label>.plist && launchctl load ~/Library/LaunchAgents/<label>.plist
```

### 查看服务日志
```bash
# 实时跟踪
tail -f <log-path>

# 最近 50 行
tail -50 <log-path>
```

### 远程机服务操作
```bash
# 查看远程机服务
ssh wj "launchctl list | grep -v com.apple"

# 重启远程机某服务
ssh wj "launchctl unload ~/Library/LaunchAgents/com.user.powerwebhook.plist && launchctl load ~/Library/LaunchAgents/com.user.powerwebhook.plist"

# 查看远程机日志
ssh wj "tail -50 ~/Library/Logs/powerwebhook.log"
```

---

## 四、系统设置「登录项与扩展 → 允许在后台」BTM 台账

> 对应「系统设置 → 通用 → 登录项与扩展」里的后台项（Background Task Management，BTM）。launchd plist 只覆盖 legacy 项，BTM 还包含 App 内嵌登录项（SMAppService）、开发商标识组与扩展，**以本节做双机后台项完整台账**。2026-09-16 实测快照。

### 4.0 取证方法（重要坑）

```bash
# 本机：直接 dump（输出约 1000~1300 行，70+ 条记录 × 系统/系统用户/登录用户三个域，每条十几个字段，行数多是正常现象）
sfltool dumpbtm

# 远程机：必须带 sudo，非交互 SSH 直接跑会报 errAuthorizationInteractionNotAllowed
# sudo 口令取法见 credentials.md 第三节，禁止明文入命令历史/日志
printf '%s\n' "$SUDOPASS" | ssh cw 'sudo -S -p "" sfltool dumpbtm'
```
- **GUI 开关关闭 ≠ legacy plist 已停用**：总开关显示「关」时，旧版 plist 的 daemon/agent 在 BTM 里仍可能是 enabled（OCLP、搜狗、HP 均有此现象），以 dump 的 Disposition 为准。
- 显示「身份不明的开发者」通常是：裸 shell/python 脚本经 launchd 拉起（无代码签名），或 App 的 root 特权助手，**不等于恶意软件**，但必须能逐条指认来源。

### 4.1 wj（.9）后台项清单（2026-09-16）

| GUI 显示名 | 状态 | 真身（Label / 路径） | 判定 |
|---|---|---|---|
| 企业微信 | 关（内嵌 IPCHelper 登录项仍开） | `com.tencent.WeWorkMac` | 正常；不用可整个卸载 |
| 搜狗输入法-语音变声斗图表情（2 项） | 总开关关，但 SogouServices / SogouTaskManager 两个 legacy agent 仍 ON | `/Library/Input Methods/SogouInput.app` | 不用语音/斗图可清，注意 legacy 残留 |
| **bash**（不明） | 开 | `com.gaodun.ep3-local-supervisor` → `~/Doubao/chats/2026-08-26/new-chat/gaodun-course-knowledge-base/scripts/ep3_local_supervisor.sh` | **自建**：高顿课程压缩+网盘上传保活总管；任务全部完成后退役 plist（见 4.4） |
| ClashX Pro（影响所有用户） | 开 | App + `com.west2online.ClashXPro.ProxyConfigHelper` root 助手 | wj 主力代理，保留 |
| CleanMyMac X（2 项） | 开 | HealthMonitor + Menu 登录项 | 常驻价值低，可关后台、用时手动开 |
| **cloudflared**（不明） | 开 | `com.user.cloudflared-power`（见 2.1） | **自建**远程关机隧道，保留 |
| **com.macpaw.CleanMyMac4.Agent**（不明） | 开 | `/Library/PrivilegedHelperTools/` 下 CleanMyMac X root 助手 | 随 CleanMyMac X 去留 |
| ~~Gemini 2~~ | **2026-09-16 已卸载**（App+支持文件共约 392MB，全部进废纸篓可恢复） | 验签发现是 **TNT 破解版**（BTM Developer Name: "TNT - why join the navy if you can be a pirate"）；BTM 残留墓碑记录在下次登录/重启后从列表消失 |
| **Hainan Youqu Technology Co., Ltd.** | 开 | **即 ToDesk 开发商「海南有趣科技」标识组**（com.youqu.todesk.*） | 随 ToDesk 去留，非陌生程序 |
| HP Device Monitor Manager / HP Inc. | 关 | `/Library/Printers/hp/...` | 无 HP 打印机可整组清 |
| HP Product Research Manager | GUI 关，`com.hp.productresearch.plist`（KeepAlive）仍在用户目录，当前未加载 | HP 产品改进遥测 | 纯遥测，建议删 plist |
| Logitech G HUB（2 项，1 影响所有用户） | 开 | ghub 用户 agent + updater 系统 daemon（`/Applications/lghub.app`） | 无罗技 G 外设建议卸（资源占用大户） |
| OCLP-Mod（2 项，1 影响所有用户） | 总开关关，但 `macos-update` daemon + `auto-patch` agent 均 ON | laobamac 版 OCLP：开机重打补丁、系统更新前准备（`/Library/Application Support/laobamac/`） | **黑苹果系统补丁，系统级，勿删/勿关** |
| **python3**（不明） | 开 | `com.user.powerwebhook`（见 2.1） | **自建**关机 Webhook，保留 |
| RustDesk | 开 | 用户 agent + 系统 service | 远程桌面，按需保留 |
| **sh**（不明） | 开 | RustDesk service 的 `/bin/sh -c .../RustDesk.app/.../service` 启动壳 | 随 RustDesk 去留 |
| Tailscale | 开 | App + 登录项助手 | 组网/VPN，保留 |
| ToDesk（2 项，1 影响所有用户） | 开 | startup/session 两个 agent + service daemon | 远程桌面，与 RustDesk/UU 三选一 |
| Tuxera Disk Manager | 标识组关，NTFS 菜单 agent 开 | `/Library/Filesystems/tuxera_ntfs.fs` | 不写 NTFS 盘可卸 |
| UU远程（2 项，1 影响所有用户） | 开 | 网易 UURemote agent + daemon | 远程桌面，三选一 |
| Xnip | 开 | App + XnipLoginHelperApp | 截图工具，保留（2026-09-16 起双机均有） |

另：「登录时打开」段（截图未含）开启的有 Aerial、Alfred 5、CheatSheet、豆包；Outlook、Parallels Desktop、Warp 等为关。共享扩展（OneNote、Tailscale、信息、发送到微信、Send with Windows Email App=Parallels 带来）均为各 App 正常注册项。

> 🔓 **2026-09-16 验签安全发现（wj，用户已知悉、暂不处理）**：PDF Expert 签名 Authority=Antibiotics（破解团队署名，确认破解版）；Parallels Desktop 为 ad-hoc 签名且内含 `CoreInject.dylib`（激活注入，配合 cw 的 PD-Runner-Helper）；Beyond Compare、CleanMyMac X 为 ad-hoc 重签（疑似破解，官方本有 Developer ID 签名）。**CleanMyMac X 的 root 特权助手常驻运行，破解重签版风险最高，优先建议换正版或卸载。** 验签命令：`codesign -dvvv /Applications/<App>.app | grep Authority`。

### 4.2 cw（.8）后台项要点（2026-09-16 sudo dump）

- **开**：RustDesk（agent + sh/service daemon）、OCLP-Mod（同 wj，补丁 daemon+agent 开、总开关显示关，**勿动**）、Clash Verge（用户 agent + `clash-verge-service` root 助手）、CleanMyMac 5 root 助手（App 本体关）、GoogleUpdater 系统 daemon、HHKBProReconnecter（PFU HHKB 键盘 kext 重连，系统级）、PD-Runner-Helper（lihaoyun6，Parallels 运行助手）、opentoken 用户服务、Alfred 5、CheatSheet、Input Source Pro。
- **关**：Tailscale App、Docker socket 标识组、Microsoft Remote Desktop Beta、Acorn、BetterZip、ScreenFlow、Xcode、企业微信、Parallels Toolbox 全套小工具（计时器/录屏/勿扰等约 20 个，均为关）。
- **Xnip**：2026-09-16 由 wj 直拷安装（v2.2.6，MAS 包），可运行、屏幕录制 TCC 授权已在；开机登录项待 App 内开关一次注册（见 ide-and-software.md 4.5）。

### 4.3 停用/删除 SOP（必须用户确认后执行）

1. 取证：`sfltool dumpbtm` 定位 Label、plist 路径、Disposition；
2. 停用：用户级 `launchctl bootout gui/$(id -u)/<label>`；系统级 `sudo launchctl bootout system/<label>`；
3. **备份式移除**（不直接 rm）：plist 移到 `~/Doubao/chats/<当日>/disabled-launchd/` 留底；系统级 plist 需 sudo；
4. App 本体按用户意愿卸载（CleanMyMac/卸载工具或删 .app）；
5. 复核：重新 dump 确认条目消失；GUI「登录项」列表同步刷新。
- 红线：OCLP-Mod、自建 cloudflared/powerwebhook/opentoken、代理客户端、Tailscale 不在清理范围；拿不准的先保留。

### 4.4 已知待办：gaodun supervisor 退役

`com.gaodun.ep3-local-supervisor` 随高顿课程任务建立（2026-08-26 项目），脚本设计为三科全部 HEVC 且上传成功后 exit 0（SuccessfulExit=false 不再重启），但 plist 仍在 `~/Library/LaunchAgents/` 且 RunAtLoad：任务收尾、尤其 `chats/2026-08-26/` 目录被清理后，bash 会因脚本不存在而反复拉起失败。**高顿任务确认结束后**：`launchctl bootout gui/$(id -u)/com.gaodun.ep3-local-supervisor` 并删除该 plist。
