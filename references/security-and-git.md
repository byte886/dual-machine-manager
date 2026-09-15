# SSH / 密钥 / 密码 / 钥匙串 + Git 工具

> **敏感原则**：本文只记录密钥路径与指纹，绝不记录私钥内容、口令、token 明文；易变版本号以现场实测为准。
> 本文只记双机 SSH/Git 侧台账与操作；加解密统一用全局命令 `secrets`（由 mac-system-toolkit 提供）。凭证分级与取密红线属上层方法论，不在本文展开、也不回指，由全局路由在安全类任务中先行加载。

---

## 一、SSH 配置对比

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| SSH 版本 | OpenSSH_9.9p2, LibreSSL 3.3.6 | OpenSSH_9.9p2, LibreSSL 3.3.6 |
| 私钥数量 | 3 把：`id_ed25519`、`id_rsa`、`id_rsa_softwawrecheng` | 3 把：同名三把（已同步） |
| 公钥数量 | 3 个 `.pub` | 3 个 `.pub` |
| **ssh-agent 状态** | ✅ 3 把常驻（launchd agent + 钥匙串） | ✅ **已修复（2026-09-15）**：launchd agent + 存入钥匙串 + `.zshenv` 持久化 SOCK，非交互 `ssh wj 'ssh-add -l'` 可见 3 把，详见第三节 |
| SSH_AUTH_SOCK | ✅ 已设（launchd listener） | ❌ 空 |
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

## 五、Git 工具对比

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| git 版本 | 2.50.1（/usr/bin Apple git） | /usr/bin 为 2.39.5 Apple git，/usr/local/bin 另有 brew git 2.55；非交互默认走 /usr/bin |
| user.name | softwarecheng | softwarecheng |
| user.email | softwarecheng@126.com | softwarecheng@126.com |
| **gh (GitHub CLI)** | ✅ `/usr/local/bin/gh`（2.58.0） | ✅ `/usr/local/bin/gh`（2.98.0；非交互 PATH 不含 /usr/local/bin 时用绝对路径） |
| lazygit / git-flow | ❌ 无 | ❌ 无 |
| Git GUI 工具 | ❌ 无（无 GitHub Desktop/Sourcetree/Tower/Fork） | ❌ 无 |
| 全局 hooksPath | 未设置 | 未设置 |
| 多账号机制 | SSH config 三账号分流 | 单账号 |

### gitconfig 差异详情

- **本机**：
  - `url.git@github.com:tinyverse-web3/.insteadof = https://github.com/tinyverse-web3/`
  - LFS 三件套已配置。
- **远程机**：
  - `safe.directory = *`
  - `credential.https://github.com.helper = !/usr/local/bin/gh auth git-credential`（gh 现已存在、配置有效；但主仓走 SSH，实际不经过该 helper）
  - `diff.submodule = log`、`status.submodulesummary = true`、`push.recursesubmodules = on-demand`、`submodule.recurse = true`

---

## 六、远程机 git credential helper（已解决，留档）

### 现状（实测）
远程机配了 `credential.https://github.com.helper = !/usr/local/bin/gh auth git-credential`（gist 同样一条）。早期 gh 未装时该 helper 指向空、HTTPS 操作会报错；**现在 gh 2.98 已装于 /usr/local/bin，helper 有效，问题已自行消除**。

### 注意
- 主仓与各技能仓都走 **SSH**（origin 为 `git@github.com`），不经过 credential helper，因此该项本就不影响提交/push。
- 非交互 SSH 的 PATH 不含 /usr/local/bin，若确需在 ssh 里调 gh，用绝对路径 `/usr/local/bin/gh`。
- 若将来改用 HTTPS 且不想用 gh helper，可改钥匙串：`git config --global credential.helper osxkeychain`。

---

## 七、GitHub 三账号 SSH 分流（仅本机配置完整）

本机 `~/.ssh/config` 按主机别名分流到不同私钥，统一走 `ssh.github.com:443`（绕过 22 端口封锁）：

| GitHub 账号 | SSH Host 别名 | 使用私钥 |
|---|---|---|
| `byte886`（主力） | `github.com` | `id_rsa_softwawrecheng` |
| tinyverse | `github-tinyverse` | `id_ed25519` |
| web3 | `github-web3` | `id_rsa` |

- **远程机**：仅主力单账号，走 `id_rsa_softwawrecheng:443`，无分流别名。
- **含义**：跨账号仓库操作只在本机做；主力账号仓库两机均可提交/push。

### gh 登录态、SSH 分流、提交身份是三件事（别混）

| 维度 | 用什么 | 账号 / 身份 | 凭据来源 |
|---|---|---|---|
| git over SSH（子模块、各仓 push/pull） | SSH 密钥 | byte886 主力（tinyverse/web3 仅本机分流） | `~/.ssh/id_rsa_*` + ssh-agent / 钥匙串 |
| git 提交署名 | `user.name` / `user.email` | softwarecheng \<softwarecheng@126.com\>，两机一致 | gitconfig，**与登录账号无关** |
| **gh（GitHub CLI：API / 建仓 / HTTPS）** | PAT | **两机统一登录为 byte886（唯一 Active）** | 加密件 `~/.doubao/secrets/github_pat.enc` 解密后 `--with-token` 登录 |

- 两机 gh 只保留 byte886 一个 Active 账号；token 已失效的历史账号用 `gh auth logout -h github.com -u <名> --force` 清掉，避免 gh 误选失效账号。
- 非交互登录 / 刷新：`secrets decrypt "$HOME/.doubao/secrets/github_pat.enc" | gh auth login -h github.com --with-token`；用 `gh auth status`、`gh api user -q .login`（应回 byte886）验证。
- 远程机非交互 SSH 下调 gh 先 `export PATH=/usr/local/bin:$PATH`（PATH 坑见 SKILL.md）。
- gh 的 "Git operations protocol" 两机可为 https/ssh 不同值——仓库实际统一走 SSH，不受影响。
- PAT 的 scope、有效期、加密落点、轮换细节见 credentials.md 第三节。

---

## 八、SSH config 其他要点

- **本机**：全局 `AddKeysToAgent yes / UseKeychain yes`；7 台云服务器；`wj` 别名 + ControlMaster 连接复用（10m）。
- **远程机**：`cw` 别名 + ControlMaster；内网服务器经 `ProxyJump root@103.103.245.177:8020` 跳转。

---

## 九、维护建议

1. ~~远程机 ssh-agent 未运行~~ **已修复（2026-09-15，见第三节）**；~~ed25519 / id_rsa 两机指纹不一致~~ **已对齐（2026-09-15 以 .8 为准、.9 旧 key 留底，见第二节）**，至此两机三把密钥完全一致。credential helper 已随 gh 安装解决（见第六节）。
2. **git 用户配置两台保持一致**（已是 softwarecheng / softwarecheng@126.com），不要在某台单独改。
3. **GitHub 多账号只在本机**，远程机不补分流，符合其单账号定位。
4. **如需 GPG 签名**：仅本机 `brew install gnupg`，远程机不跟进（其只做主力账号提交，签名策略由本机决定）。
