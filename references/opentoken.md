# OpenToken（TokenRank）专项 SOP

> 生财有术 Token 消耗排行榜本地客户端 opentoken 的安装、验证、卸载与排查全流程；版本以 `opentoken --version` 实测为准。

---

## 一、是什么

opentoken 是一个本地后台小程序，扫描本机各 AI 编程工具的 token 用量日志，按天汇总后上报到生财有术 Token 排行榜。

- **只上报 token 数量**，不上传代码或对话
- 后台服务每 30 分钟自动上报一次
- 官网：https://scys.com/tokenrank

### 当前支持的工具（v0.3.30）
Claude Code、Codex、Cline、Roo Code、WorkBuddy、CodeBuddy、CC Switch、Kilo Code、Qwen CLI、Grok CLI 等。

**注意：豆包（Doubao）当前未被官方支持**，用量数据存在 Chromium IndexedDB（LevelDB）中，格式不同。已向官方反馈，等待支持。

---

## 二、安装

### 2.1 获取安装命令
1. 打开 https://scys.com/tokenrank/connect
2. 选择系统（macOS / Linux 或 Windows）
3. 复制页面上的完整命令（含个人专属 webhook URL）

### 2.2 执行安装（macOS / Linux）
```bash
curl -fsSL https://scys.com/tokenrank/install.sh | sh -s -- "<你的-webhook-url>"
```

安装过程：
1. 下载 opentoken 二进制（约 10MB，Universal Binary 支持 x86_64 + arm64）
2. 保存配置到 `~/.opentoken/config.json`
3. 安装 launchd 服务 `com.opentoken.daemon`
4. 首次扫描并上报历史用量（可能耗时 1-5 分钟）

### 2.3 安装到远程机
```bash
ssh wj 'curl -fsSL https://scys.com/tokenrank/install.sh | sh -s -- "<你的-webhook-url>"'
```

---

## 三、安装后验证（必做）

不要只看脚本打印「完成」，必须逐项核对：

### 3.1 客户端版本
```bash
~/.local/bin/opentoken --version
# 预期输出：opentoken 0.3.30（或更新版本）
```

### 3.2 后台服务状态
```bash
launchctl list | grep opentoken
# 预期输出：-  0  com.opentoken.daemon（PID 为 - 表示当前未在运行，已加载待命）
```

### 3.3 运行频率（每 30 分钟）
```bash
grep -A1 "StartCalendarInterval" ~/Library/LaunchAgents/com.opentoken.daemon.plist
# 预期：每小时第 X 分和第 X+30 分各一次（按设备号固定错开）
# 本机示例：第 2 分、第 32 分
```

### 3.4 配置文件
```bash
cat ~/.opentoken/config.json
# 预期：包含 webhook_url 字段，值为你的专属上报地址
```

### 3.5 本地预览（不上传）
```bash
~/.local/bin/opentoken preview
# 预期：显示 Detected 工具列表和每日用量统计
```

---

## 四、常用命令

```bash
# 查看版本
~/.local/bin/opentoken --version

# 本地预览用量（不上传）
~/.local/bin/opentoken preview

# 手动触发一次上报
~/.local/bin/opentoken upload

# 手动运行一次 daemon（扫描+上报）
~/.local/bin/opentoken daemon --once

# 查看本机数据地图（落盘文件、账本规模）
~/.local/bin/opentoken data

# 查看隐私设置
~/.local/bin/opentoken privacy

# 检查自更新
~/.local/bin/opentoken self-update --check

# 应用自更新
~/.local/bin/opentoken self-update

# 服务状态
~/.local/bin/opentoken service status

# Claude Code 钩子管理
~/.local/bin/opentoken hooks install
~/.local/bin/opentoken hooks --help
```

---

## 五、文件位置

| 文件 | 路径 | 用途 |
|---|---|---|
| 二进制 | `~/.local/bin/opentoken` | 客户端程序 |
| 配置 | `~/.opentoken/config.json` | webhook URL 等 |
| 设备 ID | `~/.opentoken/device_id` | 本机稳定设备标识（随机生成，不含硬件信息） |
| 增量账本 | `~/.opentoken/state.json` | 已上报记录的内容哈希（不存原始数据） |
| v2 状态 | `~/.opentoken/v2.json` | v2 设备密钥/批次号 |
| 服务端配置缓存 | `~/.opentoken/client-config.json` | 服务端下发的采集器配置 |
| 解析缓存 | `~/.opentoken/cache/` | 本地日志解析缓存（增量扫描提速） |
| 日志 | `~/.opentoken/daemon.log` | 后台上报日志 |
| launchd plist | `~/Library/LaunchAgents/com.opentoken.daemon.plist` | 定时任务配置 |

---

## 六、卸载

### 6.1 卸载后台服务
```bash
~/.local/bin/opentoken service uninstall
```

### 6.2 删除残留文件
```bash
# 删除配置和数据
rm -rf ~/.opentoken

# 删除二进制
rm -f ~/.local/bin/opentoken

# 删除 plist（如 service uninstall 未清理）
rm -f ~/Library/LaunchAgents/com.opentoken.daemon.plist
```

### 6.3 远程机卸载
```bash
ssh wj '~/.local/bin/opentoken service uninstall && rm -rf ~/.opentoken && rm -f ~/.local/bin/opentoken && rm -f ~/Library/LaunchAgents/com.opentoken.daemon.plist'
```

### 6.4 验证卸载
```bash
# 本机
launchctl list | grep opentoken || echo "服务已移除"
ls ~/.opentoken 2>&1 || echo "配置已删除"
ls ~/.local/bin/opentoken 2>&1 || echo "二进制已删除"

# 远程机
ssh wj 'launchctl list | grep opentoken || echo "服务已移除"; ls ~/.opentoken 2>&1 || echo "配置已删除"'
```

---

## 七、常见问题排查

### 7.1 日志中出现 "another upload is running"
- **原因**：Claude Code 的 SessionEnd 钩子触发了一次上传，定时任务同时也在跑，为避免重复上报，定时任务跳过本次
- **是否正常**：正常，不影响功能
- **处理**：无需处理

### 7.2 用量无变化，无需上报
- **原因**：自上次上报以来，各工具的 token 用量没有新增
- **是否正常**：正常
- **处理**：无需处理，有新用量时会自动上报

### 7.3 某个工具的用量没被统计到
1. 运行 `~/.local/bin/opentoken preview`，查看 `Detected` 列表是否包含该工具
2. 确认该工具的日志目录存在且有数据
3. 若工具不在 Detected 列表，说明 opentoken 当前版本不支持该工具，需向官方反馈

### 7.4 服务未加载
```bash
# 重新加载
launchctl load ~/Library/LaunchAgents/com.opentoken.daemon.plist

# 或重新安装服务
~/.local/bin/opentoken service install
```

### 7.5 查看实时日志
```bash
tail -f ~/.opentoken/daemon.log
```

---

## 八、当前部署状态（以现场实测为准）

| 机器 | 状态 | 版本 | 上报时刻 |
|---|---|---|---|
| 本机 cw（192.168.2.8） | ✅ 已安装运行 | 以 `--version` 实测（盘点时 v0.3.30） | 每小时第 2/32 分 |
| 远程机 wj（192.168.2.9） | ❌ 已卸载 | — | — |

> 决策：仅保留本机安装，远程机已卸载。两台用量本会汇总到同一生财账号，目前只统计本机；若远程需要重装见第二节。
