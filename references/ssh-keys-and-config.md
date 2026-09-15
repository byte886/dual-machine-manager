# SSH 配置 / 密钥 / ssh-agent / 钥匙串 + SSH config 维护

> **敏感原则**：本文只记录密钥路径与指纹，绝不记录私钥内容、口令、token 明文；易变版本号以现场实测为准。
> 本文只记双机 SSH/密钥/ssh-agent 侧台账与操作；加解密统一用全局命令 `secrets`（由 mac-system-toolkit 提供）。凭证分级与取密红线属上层方法论，不在本文展开、也不回指。
> Git 工具对比、gh 登录态、GitHub 多账号分流见 [git-and-github-accounts.md](git-and-github-accounts.md)。

---

## 一、SSH 配置对比

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| SSH 版本 | OpenSSH_9.9p2, LibreSSL 3.3.6 | OpenSSH_9.9p2, LibreSSL 3.3.6 |
| 私钥数量 | 3 把：`id_ed25519`、`id_rsa`、`id_rsa_softwawrecheng` | 3 把：同名三把（已同步） |
| 公钥数量 | 3 个 `.pub` | 3 个 `.pub` |
| **ssh-agent 状态** | ✅ 3 把常驻（launchd agent + 钥匙串） | ✅ **已修复（2026-09-15）**：launchd agent + 存入钥匙串 + `.zshenv` 持久化 SOCK，非交互 `ssh wj 'ssh-add -l'` 可见 3 把，详见第三节 |
| SSH_AUTH_SOCK | ✅ 已设（launchd listener） | ✅ 已设（`.zshenv` 接到 launchd listener，非交互会话也有，见第三节） |
| 钥匙串 | login.keychain-db + System.keychain（默认 login） | login.keychain-db + System.keychain（默认 login） |
| GPG | ❌ 未安装 | ❌ 未安装 |
| 密码管理器 | ❌ 无 1Password/Bitwarden/KeePass | ❌ 无 |

> 注意 `id_rsa_softwawrecheng` 文件名拼写是历史遗留（softwawre），**不要改成 software**，否则 SSH config 分流会断。

---

## 二、ssh-agent 已加载密钥双机对照（仅指纹，不明文；2026-09-15 实测）

| 私钥文件 | .8 本机 cw 指纹 | .9 远程 wj 指纹 | 是否一致 |
|---|---|---|---|
| `id_rsa_softwawrecheng`（byte886 主力，git/GitHub 走它） | `5tkXaJcDPgE8Kaix7T25cj2uFnkTjbmwBJPyeeHqkhk` (RSA4096) | `5tkXaJcDPgE8Kaix7T25cj2uFnkTjbmwBJPyeeHqkhk` (RSA4096) | ✅ 一致 |
| `id_ed25519`（tinyverse） | `EZbyOb7Sgro/7W9vOGLSgeJ+xCBug+twG/doBGtv9Cs`（dev@tinyverse.space） | `EZbyOb7Sgro/7W9vOGLSgeJ+xCBug+twG/doBGtv9Cs`（dev@tinyverse.space） | ✅ 已对齐（2026-09-15） |
| `id_rsa`（web3/云服务器） | `Qk9xb0gPN1Wa02NojIIlf35RnKG7j1aW0bNJZTBC4Ko` | `Qk9xb0gPN1Wa02NojIIlf35RnKG7j1aW0bNJZTBC4Ko` | ✅ 已对齐（2026-09-15） |

> ✅ **2026-09-15 已对齐（以 .8 为准）**：此前 .9 的 ed25519（旧注释 compress-migration-20260914）/id_rsa 与 .8 不同；经内网 SSH 加密通道把 .8 两把私钥同步到 .9（私钥 600、不回显、不入库），.9 旧两把改名 `*.migrated-bak-20260915`（含 .pub）留底不删，重入 agent 与钥匙串后两机三把指纹完全一致。确认旧备份无用后可再清理。

---

## 三、远程机 ssh-agent 修复记录与通用 SOP（已于 2026-09-15 修复）

### 当时的问题根因（两层）
1. launchd 托管的 `com.openssh.ssh-agent` 进程其实在跑，但非交互/`ssh wj '<cmd>'` 会话里 `SSH_AUTH_SOCK` 为空，于是报 `Could not open a connection to your authentication agent`；
2. 三把私钥的 passphrase 从没存进该机器钥匙串，即使接上 agent 仍会交互要口令。

### 实际采用的修复（已验证，主口令不进命令行/进程参数/日志）

```bash
ssh wj
# 1) 取 launchd agent 的 listener socket（进程本就在跑，无需手动 eval ssh-agent）
SOCK=$(launchctl print gui/$(id -u)/com.openssh.ssh-agent \
  | awk '/path = \/private\/tmp.*Listeners/{print $3; exit}')

# 2) 用临时 SSH_ASKPASS 从本机 600 权限 master.pass 读 passphrase（不回显、用完即删），
#    --apple-use-keychain 同时把 passphrase 存入钥匙串，三把分别加载
ASK=$(mktemp /tmp/.askpass.XXXXXX)
printf '#!/bin/sh\ncat "$HOME/.doubao/secrets/master.pass"\n' > "$ASK"; chmod 700 "$ASK"
for k in id_rsa_softwawrecheng id_ed25519 id_rsa; do
  SSH_AUTH_SOCK="$SOCK" SSH_ASKPASS="$ASK" SSH_ASKPASS_REQUIRE=force \
    ssh-add --apple-use-keychain "$HOME/.ssh/$k"
done
rm -f "$ASK"
SSH_AUTH_SOCK="$SOCK" ssh-add -l     # 列出指纹即成功
```

### 持久化：让非交互会话也能连到 agent（关键，否则重启/新 ssh 又空）

zsh 的非交互会话不读 `.zshrc`、只读 `.zshenv`，故把下面这段写进远程机 `~/.zshenv`（幂等，已配置）：

```sh
# launchd-openssh-agent-socket
if [ -z "$SSH_AUTH_SOCK" ]; then
  _asock=$(launchctl print gui/$(id -u)/com.openssh.ssh-agent 2>/dev/null \
    | awk '/path = \/private\/tmp.*Listeners/{print $3; exit}')
  [ -S "$_asock" ] && export SSH_AUTH_SOCK="$_asock"
  unset _asock
fi
```

前提：`~/.ssh/config` 的 `Host *` 含 `AddKeysToAgent yes` 与 `UseKeychain yes`（已配）。此后 passphrase 在钥匙串、agent 由 launchd 拉起、SOCK 由 `.zshenv` 接上，三重保证免重复输；验证 `ssh wj 'ssh-add -l'` 直接列指纹、`ssh wj 'ssh -T git@github.com'` 回 `Hi byte886!`。

---

## 四、钥匙串 / GPG / 密码管理器

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| 钥匙串 | login.keychain-db + System.keychain | login.keychain-db + System.keychain |
| GPG | ❌ 未安装，Git commit 未做 GPG 签名 | ❌ 未安装，未签名 |
| 密码管理器 | ❌ 无 | ❌ 无 |

🟡 **建议**：两台均靠 macOS 钥匙串管理凭据，无独立密码管理器；如要给 Git commit 加 GPG 签名，需先 `brew install gnupg`（仅本机）并生成 GPG 密钥。

---

## 五、SSH config 其他要点

- **两机共性**：全局 `Host *` 设 `AddKeysToAgent yes / UseKeychain yes`；双机互访别名（.8 的 `wj`、.9 的 `cw`）带 ControlMaster 连接复用（10m）。
- **.8 本机**：GitHub 三账号别名（`github.com` / `github-tinyverse` / `github-web3`，见 [git-and-github-accounts.md](git-and-github-accounts.md) 第三节）+ `wj`。
- **.9 远程**：仅主力 `github.com` 单账号 + `cw`。

### 已退役主机清理台账（2026-09-16，用户确认均为旧工作残留）

下列主机**已从两机 `~/.ssh/config` 删除**，列此留痕、避免日后误以为漏配：

| 已移除主机 | 原用途 | 清理前实测 | 处理 |
|---|---|---|---|
| `39.108.96.46`（阿里云 ECS `iZwz9f1s7aknuxsng8pa9kZ`，Ubuntu 5.4 内核） | 旧公网云主机，root | 在线、两机新 key 均可登 | 已退役，删 Host |
| `103.234.53.68` | 旧公网云主机，root | 超时/无路由 | 已退役，删 Host |
| `103.103.245.177` | 旧公网跳板（.9 经其 `:8020` ProxyJump 回内网） | 超时/无路由 | 已退役，删 Host 及下游 ProxyJump |
| `192.168.10.101~104` | 旧内网网段服务器，root | 在 192.168.2.x 网段不可达 | 已退役，两机删 Host |

- 现内网统一为 **192.168.2.x（有线 en0）**：.8=192.168.2.8、.9=192.168.2.9，网关 192.168.2.1，双向 ping/SSH 通（详见 machines.md）。
- **ControlMaster 复用套接字不是垃圾**：`~/.ssh/cm-<user>@<host>:<port>`（如 `cm-wenjiechen@192.168.2.9:22`）是活动的多路复用 socket；`ssh -O check <别名>` 显示 `Master running` 即在用，勿当残留删，连接彻底关闭后按 ControlPersist 自动消失。
- **已删历史/留底文件（2026-09-16）**：.8 `config.bak-byte886-20260915-114705`；.9 `config.bak-20260914-cm`、`config.bak-agent-20260915-121111`、`known_hosts.old`，以及密钥对齐留底 `id_ed25519` / `id_rsa` 的 `*.migrated-bak-20260915`（含 .pub，云主机退役后无回退价值）。

### 备用：跳板机 ProxyJump 通用配法（当前无在用跳板，留作后用）

需要经一台公网跳板回到内网机器时，用 OpenSSH 原生 `ProxyJump`（不必手写 nc / ProxyCommand）：

```sshconfig
# 跳板机本身
Host bastion
    HostName <跳板公网 IP 或域名>
    User <跳板用户>

# 内网目标：经跳板一跳到达（多跳用逗号分隔，如 bastion,hop2）
Host internal-1
    HostName 192.168.10.101
    User root
    ProxyJump bastion        # 等价旧写法 ProxyCommand ssh -W %h:%p bastion
```

- 配好后 `ssh internal-1` 自动两段跳转；密钥/agent 自动经跳板转发，无需在跳板上落私钥。
- 跳板退役时**务必同时移除下游 Host 的 `ProxyJump` 行**，否则连接会先卡在不可达跳板上超时（本次清理的 103.103.245.177 即此情况）。

---

## 六、维护建议（SSH/密钥侧）

1. ~~远程机 ssh-agent 未运行~~ **已修复（2026-09-15，见第三节）**；~~ed25519 / id_rsa 两机指纹不一致~~ **已对齐（2026-09-15 以 .8 为准；旧 `.migrated-bak` 留底已于 2026-09-16 删除，见第二节/第五节）**，两机三把密钥完全一致。
2. **如需 GPG 签名**：仅本机 `brew install gnupg`，远程机不跟进（其只做主力账号提交，签名策略由本机决定）。
3. ~~待确认两台云主机是否退役~~ **已处理（2026-09-16）**：旧公网云主机、192.168.10 旧网段、跳板及 .9 `.migrated-bak` 旧私钥，经用户确认全部为旧工作残留并清理完毕，台账见第五节。
