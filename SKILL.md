---
name: dual-machine-manager
description: 两台 Mac（本机 cw/192.168.2.8 + 远程黑苹果 wj/192.168.2.9）的统一管家技能。覆盖系统信息、SSH 互访、launchd 后台服务、brew 服务、cron、凭证管理、OpenToken/TokenRank、远程关机、双机同步、日常巡检、故障排查等全部运维管理活；并沉淀双机深度盘点：包管理工具（Homebrew/mise/cargo/npm/pip）、IDE 与常用软件、SSH/密钥/钥匙串/Git/GitHub 多账号、机器健康度（内存/磁盘/进程/巡检）、双机差异对比与一致性维护。当用户提到「两台机器」「机器管家」「巡检」「服务状态」「后台进程」「远程关机」「双机同步」「OpenToken」「TokenRank」「wj」「黑苹果」「192.168.2.9」「凭证」「密码管理」「包管理」「Homebrew」「mise」「IDE」「VS Code 扩展」「ssh-agent」「git credential」「GitHub 多账号」「磁盘空间」「内存占用」「健康度」「双机差异」「一致性」等运维管理相关需求时使用本技能。
compatibility: macOS（已验证：macOS 15.7.8 x86_64，两台机器均为 Mac）；未验证 Windows / Linux
---

# 双机管家（Dual Machine Manager）

两台 Mac 的统一运维管理技能。所有管理信息、SOP、凭证位置均记录在 `references/` 下，按需加载。

## 平台适用（执行前先读）

- **已验证平台**：macOS 15.7.8 x86_64（本机 chenwenjie + 远程机 wenjiechen，均为 Mac）
- **未验证平台**：Windows、Linux
- **执行第一步**：`uname -s` 判平台，非 macOS 停下告知「该平台需先适配」，不用想当然的等价命令硬跑
- **路径可移植**：禁止硬编码 `/Users/<用户名>`，Shell 用 `$HOME`、Python 用 `Path.home()`、文档示例用 `~`；引号内和配置框内 `~` 不展开，用 `$HOME`

## 两台机器速览

| 别名 | 主机名 | 用户名 | IP | 定位 |
|---|---|---|---|---|
| `cw`（本机） | 192.168.2.8 | chenwenjie | 192.168.2.8 | 全能工作站，931GB，Homebrew（/usr/local） |
| `wj`（远程） | 192.168.2.9 | wenjiechen | 192.168.2.9 | 黑苹果，3.7TB，Homebrew 在 /usr/local（非交互 SSH 的 PATH 不含它，命令用绝对路径或先补 PATH） |

- SSH 互访：本机 `ssh wj` → 远程机；远程机 `ssh cw` → 本机（已互配免密 + ControlMaster）
- 双机同步：`~/Doubao` git 仓库，**两台机器都可直接提交并 push**；对端只做快进对齐（fetch → 确认无未推送提交 → `merge --ff-only` → 子模块更新），**禁止 `git pull`**，详见 references/sop.md 第二节
- **非交互 SSH 的 PATH 坑（重要）**：`ssh wj '<cmd>'` 的默认 PATH 是 `~/.cargo/bin:/usr/bin:/bin:/usr/sbin:/sbin`，**不含 `/usr/local/bin`**，所以 `ssh wj brew` 会 command not found（并非没装）。用绝对路径 `/usr/local/bin/brew`，或命令前 `export PATH=/usr/local/bin:$PATH`

## 文档索引（按需加载）

| 文档 | 何时读 | 内容 |
|---|---|---|
| [references/machines.md](references/machines.md) | 需要机器详细信息、SSH 配置、网络参数时 | 两台机器硬件/系统/网络/SSH 完整档案、互访配置、同步约定、排查命令 |
| [references/services.md](references/services.md) | 需要查看/管理后台服务、launchd、brew、cron 时 | 两台机器所有后台服务清单、服务管理通用 SOP、远程服务操作 |
| [references/credentials.md](references/credentials.md) | 需要密码、token、密钥等凭证时 | sudo 密码、SSH 密钥、GitHub PAT（byte886 主力 token 的加密落点/gh 登录/双机同步）、关机 Webhook token、OpenToken 凭证的位置与管理方式（敏感值不在这里明文存储） |
| [references/opentoken.md](references/opentoken.md) | 需要安装/验证/卸载/排查 OpenToken（TokenRank）时 | OpenToken 全流程 SOP：安装、验证（必做四项）、常用命令、文件位置、卸载、故障排查、当前部署状态 |
| [references/sop.md](references/sop.md) | 需要执行标准运维流程时 | 日常巡检、双机同步、服务管理、远程关机、故障排查、新工具接入、凭证轮换等 SOP |
| [references/package-managers.md](references/package-managers.md) | 需要对比/安装/排障包管理器时 | 双机包管理工具对比（Homebrew/npm/pip3/gem/cargo/mise/pnpm/yarn/go/java/maven/gradle）、本机 formulae 分类、远程机 Homebrew 现状与非交互 SSH 的 PATH 坑、node/npm 来源、mise 作用、一致性维护建议 |
| [references/ide-and-software.md](references/ide-and-software.md) | 需要盘点 IDE/软件时 | IDE/开发工具对比、本机 VS Code 扩展清单、远程 code CLI 现状、常用软件按分类双机对比（共有/仅本机/仅远程机） |
| [references/security-and-git.md](references/security-and-git.md) | 需要处理 SSH/密钥/钥匙串/Git/GitHub 多账号、gh 登录态时 | SSH 配置对比、本机已加载密钥指纹、远程机 ssh-agent 未运行修复 SOP、钥匙串/GPG/密码管理器状态、Git 工具与 gh 版本对比、远程机 git credential helper、GitHub 账号体系（SSH 分流 / gh 登录态 / 提交身份三者分工） |
| [references/health.md](references/health.md) | 需要看机器健康度/磁盘/内存/巡检命令时 | 资源占用对比、🔴4 个优先问题、🟡关注项、🟢健康项、磁盘详情、Home 目录大户、系统更新、只读巡检命令清单 |
| [references/comparison.md](references/comparison.md) | 需要理解双机定位/差异/一致性策略时 | 双机定位总结、全维度差异总表、必须一致/允许差异/需修复不对称、双机同步 SOP、新工具双机决策流程 |

## 常用快速操作

### 日常巡检
```bash
# 本机
ls -1 ~/Library/LaunchAgents/
launchctl list | grep -v "com.apple"
~/.local/bin/opentoken --version && launchctl list | grep opentoken
df -h / | tail -1

# 远程机
ssh wj 'ls -1 ~/Library/LaunchAgents/ && launchctl list | grep -v "com.apple" && df -h / | tail -1'
```

### 健康快检（资源/磁盘/内存/ssh-agent/git credential）
```bash
# 本机：内存压力 + 磁盘 + Home 大户
memory_pressure | tail -3; df -h /; du -sh ~/* | sort -rh | head -5

# 远程机：一次覆盖 4 个已知问题项（见 references/health.md）
ssh wj 'df -h; echo "---ssh-agent---"; ssh-add -l 2>&1; echo "---credential---"; git config --global --get credential.helper; echo "---内存---"; vm_stat | head -3'
```

### 远程关机
```bash
# sudo 密码不在仓库明文（见 references/credentials.md）；需要时由用户提供，用占位传入
ssh wj 'echo "<sudo密码>" | sudo -S shutdown -h now'
```

### OpenToken 手动上报
```bash
~/.local/bin/opentoken upload
```

### 双机同步（两机均可提交；对端只快进、禁 pull）
```bash
# 在任一台正常提交并 push
cd ~/Doubao && git add -A && git commit -m "<msg>" && git push
# 另一台对齐：先 fetch，确认本地没有未推送提交，再只快进、更新子模块（不产生合并提交）
cd ~/Doubao && git fetch origin && test -z "$(git rev-list origin/main..HEAD)" && git merge --ff-only origin/main && git submodule update --init --recursive
```

## 安全红线

1. **凭证不外露**：sudo 密码、token、密钥、webhook URL 不输出到日志、不贴到聊天、不写进非凭证文档
2. **打码义务**：对外汇报或分享时，所有 token / 密码 / 个人令牌必须打码
3. **高风险操作**：删除文件、卸载服务、关机等操作前确认范围，不主动扩展任务
4. **只读优先**：排查问题时先只读查看，不修改不删除，确认后再动手
