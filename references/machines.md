# 两台机器信息档案

> 本文档是两台 Mac 的硬件/系统/网络/SSH 基础信息记录源；易变项（占用/版本）以现场实测为准。

---

## 一、机器总览

| 项目 | 本机（主力） | 远程机（黑苹果） |
|---|---|---|
| 别名 | `cw` | `wj` |
| 主机名 | `192.168.2.8` | `192.168.2.9` |
| 用户名 | `chenwenjie` | `wenjiechen` |
| 局域网 IP | `192.168.2.8`（Wi-Fi） | `192.168.2.9`（Wi-Fi） |
| MAC 地址(en0) | `a0:36:bc:28:43:b3` | `c8:7f:54:69:eb:7b` |
| 系统 | macOS 15.7.8 (24G824) | macOS 15.7.8 (24G824) |
| 架构 | x86_64 | x86_64（黑苹果） |
| 内核 | Darwin 24.6.0 | Darwin 24.6.0 |
| 磁盘 | 931 GB（用量以 df 实测为准） | 3.7 TB（用量以 df 实测为准） |
| Homebrew | 已安装（/usr/local） | 已安装（/usr/local；非交互 SSH 的 PATH 不含它，用绝对路径） |
| git 用户 | （见 ~/.gitconfig） | softwarecheng / softwarecheng@126.com |

---

## 二、SSH 互访配置

两台机器已互配公钥免密登录，SSH config 中互为别名。

### 本机 → 远程机
- 别名：`wj`（也可直接用 IP `192.168.2.9`）
- 命令：`ssh wj`
- 配置位置：`~/.ssh/config` 中 `Host 192.168.2.9 wj`
- 关键参数：`ControlMaster auto`（多路复用，10 分钟保持）、`ServerAliveInterval 30`、`ConnectTimeout 6`

### 远程机 → 本机
- 别名：`cw`（也可直接用 IP `192.168.2.8`）
- 命令：`ssh cw`
- 配置位置：远程机 `~/.ssh/config` 中 `Host 192.168.2.8 cw`

### SSH 密钥（两台同步）
- `~/.ssh/id_rsa` — 默认 RSA 密钥（GitHub Web3Stack404 账号用）
- `~/.ssh/id_ed25519` — ED25519 密钥（GitHub tinyverse 账号用）
- `~/.ssh/id_rsa_softwawrecheng` — 主力 GitHub 账号 byte886 用（注意文件名拼写 softwawrecheng，是历史拼写，不要改）

### SSH config 中的其他 Host
两台机器的 config 中还配置了多台云服务器（root 用户）：
- `103.234.53.68`、`103.103.245.177`、`39.108.96.46`（公网云服务器）
- `192.168.10.101` ~ `192.168.10.104`（局域网服务器）
- GitHub 多账号别名：`github.com`（byte886 主力）、`github-tinyverse`、`github-web3`

---

## 三、双机同步约定

根据 `~/Doubao/AGENTS.md`：
- 同一套 `~/Doubao` 在两台 Mac 间同步，**两台均可提交并 push**，对端按 `fetch → 确认无未推送提交（rev-list 为空）→ merge --ff-only → submodule update` 快进对齐，禁止 pull（详见 dual-machine-manager/references/sop.md 第二节）
- 技能与脚本内**禁止硬编码 `/Users/<用户名>`**：Shell 用 `$HOME`、Python 用 `Path.home()`、文档示例用 `~`
- 引号内和 MCP/GUI 配置框内 `~` 不展开，这类位置用 `$HOME`

---

## 四、快速排查命令

```bash
# 测试远程机连通性
ssh -o ConnectTimeout=6 wj "echo OK"

# 查看远程机系统信息
ssh wj "sw_vers && uname -m && hostname"

# 查看远程机磁盘
ssh wj "df -h /"

# 从本机复制文件到远程机
scp /path/to/local/file wj:/path/to/remote/

# 从远程机复制文件到本机
scp wj:/path/to/remote/file /path/to/local/
```
