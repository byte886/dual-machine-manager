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
