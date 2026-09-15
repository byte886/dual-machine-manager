# 密码与凭证管理

> 本文档记录两台机器的凭证位置、管理方式与使用规范。本仓库为公开仓，**敏感值一律不写明文**，只记录位置与获取方式。

---

## 一、系统密码

### sudo 密码
- **两台机器使用同一个统一主密码**（真实值不写入公开仓库；需要时由用户当面提供，或从用户本机密码保管处取用）
- **使用场景**：需要 sudo 权限的命令行操作
- **免交互写法**：`echo '<sudo密码>' | sudo -S <command>`，密码以占位/变量传入，**禁止把真实值写进任何入库脚本或文档、不在日志回显**；仅限可信脚本

---

## 二、SSH 密钥

两台机器的 SSH 密钥完全同步，共三把：

| 密钥文件 | 类型 | 用途 |
|---|---|---|
| `~/.ssh/id_rsa` | RSA | GitHub Web3Stack404 账号；云服务器默认登录 |
| `~/.ssh/id_ed25519` | ED25519 | GitHub tinyverse 账号 |
| `~/.ssh/id_rsa_softwawrecheng` | RSA | GitHub 主力账号 byte886（Doubao 工作仓）；注意文件名拼写 `softwawrecheng` 是历史拼写，不要改 |

### 密钥管理
- 所有私钥已加入 ssh-agent，macOS 钥匙串保存口令，首次输入后长期免输
- SSH config 中 `Host *` 配置了 `AddKeysToAgent yes` 和 `UseKeychain yes`
- GitHub 多账号通过 `IdentitiesOnly yes` 隔离，避免误认证

### 公钥分发
- 两台机器互配公钥免密（`ssh wj` / `ssh cw` 无需密码）
- 多台云服务器（root）已配置公钥登录

---

## 三、GitHub Personal Access Token (PAT)

### 存储方式
- **加密存储**，不明文保存
- **解密主密码**：与系统统一主密码相同（值不入库，需要时由用户提供）
- **常用话术**：「自动化获取 playwright token」（用户已记录为常用话术，触发时自动解密获取）

### 使用场景
- 需要 GitHub API 认证的自动化脚本
- Playwright 等工具的 token 获取

---

## 四、远程关机 Webhook Token（远程机 wj）

### 位置
- 存储在远程机 launchd plist 的环境变量中：
  `~/Library/LaunchAgents/com.user.powerwebhook.plist` → `SHUTDOWN_TOKEN`

### 用途
- 认证远程关机 HTTP 请求
- 配合 cloudflared tunnel（`power-webhook`）暴露到公网

### 安全注意
- 此 token 等同于远程关机权限，**不要对外分享**
- 查看方式：`ssh wj "grep -A1 SHUTDOWN_TOKEN ~/Library/LaunchAgents/com.user.powerwebhook.plist"`
- 如需轮换：修改 plist 中的 `SHUTDOWN_TOKEN` 值，然后 `launchctl unload` + `load` 重启服务

---

## 五、OpenToken 接入凭证（TokenRank）

### 位置
- 本机：`~/.opentoken/config.json` → `webhook_url`
- 凭证形式：URL 路径中包含个人令牌（`/api/subapp/u/<令牌>`）

### 安全注意
- 此 URL 绑定生财有术账号，**是专属凭证，请勿分享给他人或公开截图**
- 他人获取后可冒用名义上报 token 用量
- 汇报或贴日志时，接入地址（含个人令牌）必须打码

### 查看（已打码示例）
```bash
python3 -c "import json; c=json.load(open('$HOME/.opentoken/config.json')); u=c['webhook_url']; print(u[:50]+'...'+u[-12:])"
```

---

## 六、凭证管理原则

1. **最小暴露**：敏感值不写进文档、不输出到日志、不贴到聊天
2. **位置优先**：文档只记录凭证存在哪里、怎么获取，不记录值本身
3. **打码义务**：对外汇报或分享时，所有 token / 密码 / 密钥必须打码
4. **轮换机制**：怀疑泄露时立即轮换（webhook token、PAT、关机 token 均可重新生成）
5. **双机同步**：SSH 密钥两台机器保持一致；`~/Doubao` 两台均可提交，对端只快进对齐（禁 pull，见 sop.md 第二节）
