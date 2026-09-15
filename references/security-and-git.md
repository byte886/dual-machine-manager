# SSH / 密钥 / 密码 / 钥匙串 + Git 工具

> **敏感原则**：本文只记录密钥路径与指纹，绝不记录私钥内容、口令、token 明文；易变版本号以现场实测为准。
> **方法论指针**：凭证分级、AI/开发取密 SOP、主口令怎么向用户要、公开仓红线统一看 `skills/security-baseline/`；加解密命令看 mac-system-toolkit。本文只记双机 SSH/Git 侧的台账与操作，不重复其条文。

---

## 一、SSH 配置对比

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| SSH 版本 | OpenSSH_9.9p2, LibreSSL 3.3.6 | OpenSSH_9.9p2, LibreSSL 3.3.6 |
| 私钥数量 | 3 把：`id_ed25519`、`id_rsa`、`id_rsa_softwawrecheng` | 3 把：同名三把（已同步） |
| 公钥数量 | 3 个 `.pub` | 3 个 `.pub` |
| **ssh-agent 状态** | ✅ **3 把密钥已加载**（2026-09-15 实测） | 🔴 **加载 0 把、仍待修**（2026-09-15 `ssh-add -l` 实测，修复见第三节） |
| SSH_AUTH_SOCK | ✅ 已设（launchd listener） | ❌ 空 |
| 钥匙串 | login.keychain-db + System.keychain（默认 login） | login.keychain-db + System.keychain（默认 login） |
| GPG | ❌ 未安装 | ❌ 未安装 |
| 密码管理器 | ❌ 无 1Password/Bitwarden/KeePass | ❌ 无 |

> 注意 `id_rsa_softwawrecheng` 文件名拼写是历史遗留（softwawre），**不要改成 software**，否则 SSH config 分流会断。

---

## 二、本机 ssh-agent 已加载密钥（仅指纹，不明文）

```
4096 SHA256:5tkXaJcDPgE8Kaix7T25cj2uFnkTjbmwBJPyeeHqkhk  softwawrecheng@github (RSA)
256  SHA256:EZbyOb7Sgro/7W9vOGLSgeJ+xCBug+twG/doBGtv9Cs  dev@tinyverse.space (ED25519)
4096 SHA256:Qk9xb0gPN1Wa02NojIIlf35RnKG7j1aW0bNJZTBC4Ko  softwarecheng@126.com (RSA)
```

---

## 三、🔴 远程机 ssh-agent 未运行 — 修复 SOP

### 问题
远程机 `SSH_AUTH_SOCK` 为空，agent 未运行（`Could not open a connection to your authentication agent`）。当前免密依赖 ControlMaster 复用连接；但 agent 重启或新连接时，私钥口令无法走钥匙串自动注入，可能突然要求输密码。

### 修复步骤

```bash
ssh wj
# 1. 启动 agent 并把已同步的三把私钥加载进钥匙串
eval $(ssh-agent)
ssh-add --apple-use-keychain ~/.ssh/id_rsa_softwawrecheng ~/.ssh/id_ed25519 ~/.ssh/id_rsa

# 2. 验证
ssh-add -l          # 应列出 3 把密钥指纹
echo $SSH_AUTH_SOCK # 应非空
```

### 配置 launchd 开机自启（避免重启后又丢）

在远程机创建 `~/Library/LaunchAgents/` 下的 plist，让 ssh-agent 随登录启动（macOS 上 ssh-agent 通常由 launchd 自动拉起 listener；若未拉起，需确认 `~/.ssh/config` 含：）

```
AddKeysToAgent yes
UseKeychain yes
```

并把密钥口令存入钥匙串后，开机即可自动加载，无需每次手动 `ssh-add`。

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

1. **优先修远程机 ssh-agent**（第三节）——这是当前唯一仍存在的不对称项；credential helper 已随 gh 安装解决（见第六节）。
2. **git 用户配置两台保持一致**（已是 softwarecheng / softwarecheng@126.com），不要在某台单独改。
3. **GitHub 多账号只在本机**，远程机不补分流，符合其单账号定位。
4. **如需 GPG 签名**：仅本机 `brew install gnupg`，远程机不跟进（其只做主力账号提交，签名策略由本机决定）。
