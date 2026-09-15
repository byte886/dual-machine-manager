# 两台机器信息档案

> 本文档是两台 Mac 的硬件/系统/网络/SSH 基础信息记录源；易变项（占用/版本/进程）以现场实测为准。
> 硬件型号 2026-09-15 由 `system_profiler SPHardwareDataType` / `sysctl machdep.cpu.brand_string` / `df -h` 实测；**设备序列号等唯一标识属敏感信息，不写入公开仓**。

---

## 一、机器总览

| 项目 | 本机（主力 cw） | 远程机（黑苹果 wj） |
|---|---|---|
| 别名 | `cw` | `wj` |
| 主机名 | `192.168.2.8` | `192.168.2.9` |
| 用户名 | `chenwenjie` | `wenjiechen` |
| 局域网 IP | `192.168.2.8`（Wi-Fi/有线，以现场为准） | `192.168.2.9`（以现场为准） |
| MAC 地址(en0) | `a0:36:bc:28:43:b3` | `c8:7f:54:69:eb:7b` |
| 机型标识（SMBIOS） | Mac Pro（**MacPro7,1，黑苹果伪装机型**） | Mac Pro（**MacPro7,1，黑苹果伪装机型**） |
| 系统 | macOS 15.7.8 (24G824) | macOS 15.7.8 (24G824) |
| 架构/内核 | x86_64 / Darwin 24.6.0 | x86_64（黑苹果）/ Darwin 24.6.0 |
| Homebrew | 已安装（/usr/local） | 已安装（/usr/local；非交互 SSH 的 PATH 不含它，用绝对路径） |
| git 用户 | softwarecheng / softwarecheng@126.com（以 ~/.gitconfig 为准） | softwarecheng / softwarecheng@126.com |

---

## 二、硬件配置详表（2026-09-15 实测）

> 两台均为 x86 黑苹果（非 Apple 原生芯片），SMBIOS 统一伪装成 MacPro7,1；下列为实测值，占用/可用量以现场命令为准。

| 硬件项 | 本机 cw（全能工作站） | 远程机 wj（编译/服务器） |
|---|---|---|
| CPU | 12th Gen Intel **i5-12600K** @3.68GHz，10 核（1 颗） | 13th Gen Intel **i5-13600KF** @3.5GHz，14 核（1 颗，无核显） |
| 内存 | **64 GB** | **128 GB** |
| 显卡 | AMD Radeon **RX 460/560，2GB VRAM** | AMD Radeon **RX 460/560，4GB VRAM** |
| 系统盘 | 931GB NVMe（根卷 `/`，实测可用约 477GB） | 3.7TB NVMe（根卷 `/`，数据卷可用约 2.2TB） |
| 额外数据盘 | — | `/Volumes/backup` 15TB、`/Volumes/sys` 194GB、`/Volumes/s` 759GB、`/Volumes/新加卷` 829MB（多盘/多系统） |
| 黑苹果引导 | OpenCore（配套 OpenCore Configurator / OCLP-Mod，见 ide-and-software.md） | 同左，另装 Macs Fan Control 散热监控 |

实测命令（只读，可随时复核）：

```bash
system_profiler SPHardwareDataType      # 机型/CPU/内存（序列号不要写进仓库）
sysctl -n machdep.cpu.brand_string      # CPU 型号全称
system_profiler SPDisplaysDataType      # 显卡/VRAM
df -h                                    # 磁盘与可用量
```

> 资源实时占用（内存压力、磁盘大户、健康灯）见 [health.md](health.md)；全维度差异与"必须一致/允许差异"策略见 [comparison.md](comparison.md)。

---

## 三、SSH 互访配置

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

> ssh-agent 加载状态、钥匙串、GitHub 多账号分流见 [security-and-git.md](security-and-git.md)；凭证"来处"与取密方法论见 security-baseline 技能。

### SSH config 中的其他 Host
两台机器的 config 中还配置了多台云服务器（root 用户）：
- `103.234.53.68`、`103.103.245.177`、`39.108.96.46`（公网云服务器）
- `192.168.10.101` ~ `192.168.10.104`（局域网服务器）
- GitHub 多账号别名：`github.com`（byte886 主力）、`github-tinyverse`、`github-web3`

---

## 四、双机同步约定

根据 `~/Doubao/AGENTS.md`：
- 同一套 `~/Doubao` 在两台 Mac 间同步，**两台均可提交并 push**，对端按 `fetch → 确认无未推送提交（rev-list 为空）→ merge --ff-only → submodule update` 快进对齐，禁止 pull（详见 dual-machine-manager/references/sop.md 第二节）
- 技能与脚本内**禁止硬编码 `/Users/<用户名>`**：Shell 用 `$HOME`、Python 用 `Path.home()`、文档示例用 `~`
- 引号内和 MCP/GUI 配置框内 `~` 不展开，这类位置用 `$HOME`

---

## 五、快速排查命令

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
