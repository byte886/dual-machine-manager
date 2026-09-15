# 密码与凭证管理（双机台账）

> 本篇只记两台机器**实际有哪些凭证、放在哪、怎么取、双机是否一致**这些台账事实，不承载凭证分级、AI/开发取密 SOP、公开仓红线等方法论——那些是上层规矩，由全局路由（`~/Doubao/AGENTS.md`）在命中安全类任务时先行加载，本篇不回指、不复制，避免双向循环。本仓为公开仓，**敏感值一律不写明文**，只记位置与获取方式。

---

## 一、"需要密码/凭证的工具"总览矩阵

| 工具/场景 | 需要什么凭证 | 凭证来处 | 取用方式 / SOP |
|---|---|---|---|
| `gh`（建仓/API/PR/Release） | GitHub PAT（byte886，classic 全权限、永不过期） | 全局 `~/.doubao/secrets/github_pat.enc` | 解密管道 `gh auth login --with-token`，见第五节；遵循全局取—用—弃纪律（解密进变量、不回显、用完 unset） |
| `git push`（SSH） | SSH 私钥 + passphrase | `~/.ssh/id_*` + ssh-agent/钥匙串 | `ssh-add --apple-use-keychain` 一次后免输，详见 [ssh-keys-and-config.md](ssh-keys-and-config.md) |
| `sudo` / 系统级命令 | 统一主口令 | 交互输入（优先）；.9 另有 `sudo.enc` 供自动化 | 见第三节；不让 agent 经手时由用户终端亲输 |
| 远程关机（webhook） | `SHUTDOWN_TOKEN` | 远程机 launchd plist | 见第六节 / [sop.md](sop.md) 第五节 |
| OpenToken/TokenRank 上报 | webhook_url（含个人令牌） | `~/.opentoken/config.json` | 见第七节 / [opentoken.md](opentoken.md) |
| 脚本调第三方平台 | API key/secret | 项目 `<项目>/.secrets/*.enc` | `secrets` 运行时解密进内存，明文不入库 |
| 二段因子登录（2FA） | TOTP 6 位动态码 | **用户手机 Microsoft Authenticator** | 当次向用户要、用后即弃，agent 默认不持有 TOTP secret |
| 加解密本身（`secrets` 命令） | 统一主口令 | 用户交互提供；.9 无人值守用 `master.pass` | 主口令只在当次内存用于解密、不回显不入库 |

> 一句话：**外部服务只看到各自不同的 token/key，直接间接都拿不到用户主口令明文**；这是整套设计的目标。

## 二、双机加密凭证清单（2026-09-15 实测台账）

目录 `~/.doubao/secrets/`（权限 700，仓库外、**永不入库**），文件权限 600：

| 文件 | .8 本机 cw | .9 远程 wj | 用途 |
|---|---|---|---|
| `github_pat.enc` | ✅ 90B | ✅ 90B | byte886 主力 PAT（双机同密文、可互拷） |
| `sudo.enc` | ❌ 无（交互为主） | ✅ 45B | 无人值守 sudo 的加密口令 |
| `master.pass` | ❌ 无（默认交互给主口令） | ✅ 8B（600） | 非交互解密用本机主口令文件，仅可信本机、不入云同步 |

- 两机不对称是**刻意结果**：.8 以交互为主、不需要落 master.pass；.9 承担无人值守自动化才配 `sudo.enc`/`master.pass`。
- 复核命令（只列文件名/权限，不读内容）：`ls -la ~/.doubao/secrets/`
- 项目级凭证在各项目 `<项目>/.secrets/*.enc`（`.enc` 可随仓、明文不入库），不在本全局清单。
- 变更后双机对齐方式见第八节与 [sop.md](sop.md) 第八节。

---

## 三、系统密码（sudo）

- **两台机器使用同一个统一主口令**（真实值不写入公开仓库；需要时由用户当面/交互提供，或从 `.enc` 解密）。
- **使用场景**：需要 sudo 权限的命令行操作。
- **优先让用户在自己终端亲输**；确需自动化时：
  - .9 用 `~/.doubao/secrets/sudo.enc`；.8 无该文件，按需再建。
  - `echo '<sudo密码>' | sudo -S <command>`，密码以占位/变量传入，**禁止真实值进入库脚本/文档、不在日志回显**，仅限可信脚本。

---

## 四、SSH 密钥（摘要，详见 ssh-keys-and-config.md）

两台同步共三把：`id_rsa`（Web3Stack404）、`id_ed25519`（tinyverse）、`id_rsa_softwawrecheng`（byte886 主力，文件名历史拼写勿改）。

- 所有私钥应交由 ssh-agent + macOS 钥匙串，首次输 passphrase 后免重复输入。
- **实测状态（2026-09-15）**：两机 ssh-agent 均加载 3 把，且三把密钥指纹两机完全一致（.9 当日修复 agent，并以 .8 为准对齐 ed25519/id_rsa、旧 key 改名 `.migrated-bak` 留底，见 [ssh-keys-and-config.md](ssh-keys-and-config.md) 第二、三节）。
- 双机互配公钥免密（`ssh wj` / `ssh cw`），2026-09-16 双向实测通过；GitHub 三账号公钥均有效（byte886/tinyverse/Web3Stack404）。旧公网云服务器（root）已退役、两机 SSH config 已清，`.9` 密钥对齐时的 `.migrated-bak` 旧私钥留底也已删除。

---

## 五、GitHub Personal Access Token（PAT）

### 当前主力 PAT（账号 byte886）
- **类型/有效期**：classic token，**No expiration（永不过期）**，供长期自动化；账号级全权限（21 个顶层 scope，页面子权限被父权限隐含，最终等价全选，属正常）。
- **加密落点（双机同路径、仓库外、永不入库）**：`~/.doubao/secrets/github_pat.enc`（两机均 90B，统一主口令一致、密文可互拷；目录 700、文件 600）。
- **加解密工具**：全局命令 `secrets`（由 mac-system-toolkit 安装提供；算法 aes-256-cbc + pbkdf2 + base64 固定不改，保证双机/新旧密文互解）。

### 取用与 gh 登录（标准命令，取—用—弃）
```bash
TOKEN="$(ENC_PASS='<统一主口令>' secrets decrypt "$HOME/.doubao/secrets/github_pat.enc")"
printf '%s' "$TOKEN" | gh auth login -h github.com --with-token
gh api user -q .login            # 应回 byte886；不打印 token
unset TOKEN
```
- token 不进 git/脚本/shell 历史；现解密、仅变量短暂持有、用完 unset；对外最多显示 `ghp_xxxx…后4位`。
- 主口令与系统统一口令相同（值不入库，交互提供，或读 .9 的 `master.pass`）。
- **常用话术**：「自动化获取 playwright token」（用户已记录，触发时按本节自动解密获取，过程不回显）。

### 双机同步与轮换
- PAT 更新后**两台都要改**：新 `github_pat.enc` 经 scp 送对端同路径，各自重跑 `gh auth login --with-token`；旧 token 到 GitHub 网页删除退役。
- 完整轮换步骤见 [sop.md](sop.md) 第八节；新建 GitHub 仓默认 **public**（用户硬偏好）。

---

## 六、远程关机 Webhook Token（仅远程机 wj）

- 位置：远程机 `~/Library/LaunchAgents/com.user.powerwebhook.plist` → `SHUTDOWN_TOKEN`，配合 cloudflared tunnel（`power-webhook`）暴露公网。
- 等同于远程关机权限，**不对外分享**；查看：`ssh wj "grep -A1 SHUTDOWN_TOKEN ~/Library/LaunchAgents/com.user.powerwebhook.plist"`。
- 轮换：改 plist 值后 `launchctl unload` + `load` 重启服务。

---

## 七、OpenToken 接入凭证（TokenRank）

- 本机 `~/.opentoken/config.json` → `webhook_url`，URL 路径含个人令牌（`/api/subapp/u/<令牌>`）。
- 绑定生财有术账号，**专属凭证、不分享、公开截图必须打码**；他人拿到可冒用名义上报。
- 打码查看：
```bash
python3 -c "import json,os; c=json.load(open(os.path.expanduser('~/.opentoken/config.json'))); u=c['webhook_url']; print(u[:50]+'...'+u[-12:])"
```

---

## 八、凭证管理原则

1. **位置优先于值**：只记"凭证在哪、怎么取"，不记值本身；分级与取密红线属上层方法论，不在本篇展开。
2. **最小暴露**：不写进非凭证文档、不输出到日志、不贴到聊天；对外一律打码。
3. **加密落盘**：全局 `~/.doubao/secrets/*.enc`（永不入库）、项目 `.secrets/*.enc`（仅密文可随仓），明文绝不入库。
4. **轮换机制**：怀疑泄漏立即轮换；轮换后更新本台账位置说明、双机对齐、验证新凭证、退役旧凭证（SOP 见 sop.md 第八节）。
5. **双机一致性**：SSH 密钥、`github_pat.enc` 两台保持一致；`sudo.enc`/`master.pass` 按是否需要无人值守差异化保留（见第二节）。
