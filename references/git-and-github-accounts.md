# Git 工具 / credential helper / GitHub 多账号分流

> **敏感原则**：本文只记配置与登录态台账，绝不记录 token 明文、私钥内容；易变版本号以现场实测为准。
> 本文只记双机 Git/GitHub 侧台账；加解密统一用全局命令 `secrets`（由 mac-system-toolkit 提供）。凭证分级与取密红线属上层方法论，不在本文展开、也不回指。
> SSH 配置、密钥指纹、ssh-agent、钥匙串见 [ssh-keys-and-config.md](ssh-keys-and-config.md)。

---

## 一、Git 工具对比

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

## 二、远程机 git credential helper（已解决，留档）

### 现状（实测）
远程机配了 `credential.https://github.com.helper = !/usr/local/bin/gh auth git-credential`（gist 同样一条）。早期 gh 未装时该 helper 指向空、HTTPS 操作会报错；**现在 gh 2.98 已装于 /usr/local/bin，helper 有效，问题已自行消除**。

### 注意
- 主仓与各技能仓都走 **SSH**（origin 为 `git@github.com`），不经过 credential helper，因此该项本就不影响提交/push。
- 非交互 SSH 的 PATH 不含 /usr/local/bin，若确需在 ssh 里调 gh，用绝对路径 `/usr/local/bin/gh`。
- 若将来改用 HTTPS 且不想用 gh helper，可改钥匙串：`git config --global credential.helper osxkeychain`。

---

## 三、GitHub 三账号 SSH 分流（仅本机配置完整）

本机 `~/.ssh/config` 按主机别名分流到不同私钥，统一走 `ssh.github.com:443`（绕过 22 端口封锁）：

| GitHub 账号 | SSH Host 别名 | 使用私钥 |
|---|---|---|
| `byte886`（主力） | `github.com` | `id_rsa_softwawrecheng` |
| tinyverse | `github-tinyverse` | `id_ed25519` |
| web3 | `github-web3` | `id_rsa` |

- **远程机**：仅主力单账号，走 `id_rsa_softwawrecheng:443`，无分流别名。
- **含义**：跨账号仓库操作只在本机做；主力账号仓库两机均可提交/push。
- **2026-09-16 实测三别名均有效**：`github.com`→byte886、`github-tinyverse`→tinyverse、`github-web3`→Web3Stack404 全部认证成功，三把私钥在 GitHub 侧均有效，无失效 key。

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

## 四、维护建议（Git/GitHub 侧）

1. credential helper 已随 gh 安装解决（见第二节）。
2. **git 用户配置两台保持一致**（已是 softwarecheng / softwarecheng@126.com），不要在某台单独改。
3. **GitHub 多账号只在本机**，远程机不补分流，符合其单账号定位。
