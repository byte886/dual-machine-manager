# 标准操作流程（SOP）总览

> 两台机器日常维护的标准操作流程。每个 SOP 都是可直接执行的步骤清单。

---

## 一、日常巡检 SOP

**触发**：用户说「检查一下机器状态」「巡检」「看看服务跑没跑」

### 步骤
1. **本机服务状态**
   ```bash
   echo "=== launchd 用户服务 ===" && ls -1 ~/Library/LaunchAgents/
   echo "=== 已加载服务(非系统) ===" && launchctl list | grep -v "com.apple"
   echo "=== OpenToken 状态 ===" && ~/.local/bin/opentoken --version && launchctl list | grep opentoken
   echo "=== 磁盘 ===" && df -h / | tail -1
   ```

2. **远程机服务状态**
   ```bash
   ssh wj 'echo "=== launchd 用户服务 ===" && ls -1 ~/Library/LaunchAgents/ && echo "=== 已加载服务(非系统) ===" && launchctl list | grep -v "com.apple" && echo "=== 磁盘 ===" && df -h / | tail -1'
   ```

3. **OpenToken 用量预览**
   ```bash
   ~/.local/bin/opentoken preview
   ```

4. **汇报结果**：列出每台机器的服务状态、磁盘、异常项

---

## 二、双机同步 SOP（~/Doubao）

**触发**：用户说「同步两台机器」「推送 Doubao」「git 同步」

### 背景与原则
- `~/Doubao` 是两台机器共享的工作区（README、PROFILE、AGENTS、skills、chats）
- **两台机器都可以直接提交并 push**，不再限定某一台；远程机 wj 不是唯一提交点
- **对端只做快进（ff-only）对齐，禁止 `git pull`、禁止产生合并提交**；子模块改动遵循"先子后父"

### 步骤
1. **当前机：确认工作区后提交推送**
   ```bash
   cd ~/Doubao && git status -s          # 有在途改动先列给用户，不擅自丢弃
   git add -A && git commit -m "<提交信息>" && git push
   ```
   若含子模块（`skills/<仓>`）：先进子仓 commit/push，再回主仓 `git add skills/` 提交（已配 push.recurseSubmodules=on-demand）。

2. **另一台：快进对齐（不要用 pull）**
   ```bash
   cd ~/Doubao
   git fetch origin
   git rev-list origin/main..HEAD       # 必须为空（本地无未推送提交）才继续，否则先处理在途提交
   git merge --ff-only origin/main
   git submodule update --init --recursive
   ```
   经 ssh 在远程机执行时，其非交互 PATH 不含 /usr/local/bin；对齐只用系统 git（/usr/bin）即可，无需 brew 版。

3. **注意事项**
   - 技能和脚本内禁止硬编码 `/Users/<用户名>`，用 `$HOME` / `Path.home()` / `~`
   - 引号内和配置框内 `~` 不展开，用 `$HOME`
   - 第三方源码和构建产物不手改

---

## 三、服务管理 SOP

### 3.1 重启本机 launchd 服务
```bash
launchctl unload ~/Library/LaunchAgents/<label>.plist
launchctl load ~/Library/LaunchAgents/<label>.plist
```

### 3.2 重启远程机 launchd 服务
```bash
ssh wj 'launchctl unload ~/Library/LaunchAgents/<label>.plist && launchctl load ~/Library/LaunchAgents/<label>.plist'
```

### 3.3 查看服务日志
```bash
# 本机
tail -50 ~/.opentoken/daemon.log

# 远程机
ssh wj 'tail -50 ~/Library/Logs/powerwebhook.log'
ssh wj 'tail -50 ~/Library/Logs/cloudflared-power.log'
```

### 3.4 新增 launchd 服务
1. 编写 plist 文件到 `~/Library/LaunchAgents/<label>.plist`
2. `launchctl load ~/Library/LaunchAgents/<label>.plist`
3. 验证：`launchctl list | grep <label>`

---

## 四、OpenToken 管理 SOP

详见 [opentoken.md](opentoken.md)，常用操作：

```bash
# 安装
curl -fsSL https://scys.com/tokenrank/install.sh | sh -s -- "<webhook-url>"

# 验证（必做四项）
~/.local/bin/opentoken --version
launchctl list | grep opentoken
grep -A1 "StartCalendarInterval" ~/Library/LaunchAgents/com.opentoken.daemon.plist
~/.local/bin/opentoken preview

# 手动上报
~/.local/bin/opentoken upload

# 卸载
~/.local/bin/opentoken service uninstall
rm -rf ~/.opentoken ~/.local/bin/opentoken ~/Library/LaunchAgents/com.opentoken.daemon.plist
```

---

## 五、远程关机 SOP

**触发**：用户说「关掉黑苹果」「远程关机」「关 wj」

### 方式一：通过 Webhook（需 cloudflared tunnel 在线）
远程机运行着 `powerwebhook` 服务，通过 cloudflared tunnel 暴露到公网。发送带 token 的 HTTP 请求即可关机。

> token 获取见 [credentials.md](credentials.md) 第四节。

### 方式二：通过 SSH（推荐，更可靠）
```bash
ssh wj 'echo "<sudo密码，见 credentials.md>" | sudo -S shutdown -h now'
```

### 方式三：SSH 后手动
```bash
ssh wj
sudo shutdown -h now
```

---

## 六、故障排查 SOP

### 6.1 SSH 连不上远程机
1. **测试网络连通性**：`ping 192.168.2.9`
2. **测试 SSH 端口**：`nc -zv 192.168.2.9 22`
3. **详细模式连接**：`ssh -v wj` 看卡在哪一步
4. **常见原因**：远程机关机/睡眠、IP 变化、SSH 服务未启动

### 6.2 服务不工作
1. 查看服务状态：`launchctl list | grep <label>`
2. 查看 plist：`cat ~/Library/LaunchAgents/<label>.plist`
3. 查看日志：`tail -50 <log-path>`
4. 手动运行程序看报错：直接执行 plist 中的 ProgramArguments
5. 重启服务：unload + load

### 6.3 磁盘空间不足
```bash
# 查看大文件
du -sh ~/* | sort -rh | head -20

# 清理缓存
rm -rf ~/Library/Caches/*
```

---

## 七、新工具/新服务接入 SOP

当需要在机器上安装新的后台工具或服务时：

1. **记录到 services.md**：服务名、用途、plist 路径、日志路径、管理命令
2. **记录到 credentials.md**（如涉及凭证）：凭证位置、获取方式、安全注意
3. **更新本文档**：如涉及新的操作流程，补充对应 SOP
4. **双机同步**：如涉及 `~/Doubao` 变更，按双机同步 SOP 提交
5. **验证**：安装后按验证清单逐项确认

---

## 八、凭证轮换 SOP

当怀疑凭证泄露或定期轮换时：

| 凭证 | 轮换方式 |
|---|---|
| sudo 密码 | 系统设置 → 用户与群组 → 修改密码；两台机器分别改 |
| SSH 密钥 | `ssh-keygen` 生成新密钥 → 更新 `~/.ssh/config` → 分发公钥到所有服务器 → 退役旧密钥 |
| GitHub PAT | GitHub Settings → Developer settings → Personal access tokens → 生成新 token → 更新加密存储 → 退役旧 token |
| 关机 Webhook token | 编辑远程机 plist 中的 `SHUTDOWN_TOKEN` → unload + load 重启服务 |
| OpenToken webhook | 重新从官网获取安装命令 → 重新执行安装（覆盖配置） |

轮换后必须：更新 credentials.md 中的位置说明、验证新凭证可用、退役旧凭证。
