# 双机差异对比与一致性维护

> 本文是双机定位、全维度差异、一致性策略与同步 SOP 的总览；细节见各专题 reference。
> 版本号 / 包数量 / 资源占用等为易变项，以现场实测为准，本文只保留稳定结构与结论。

---

## 一、双机定位总结

| | 本机 cw | 远程机 wj |
|---|---|---|
| **定位** | **全能工作站**：主力开发 + 设计 + 日常办公，工具链最全 | **黑苹果编译/服务器**：编译、多系统、散热与 AI 开发任务，工具链精简 |
| 用户名 | chenwenjie | wenjiechen |
| IP | 192.168.2.8 | 192.168.2.9 |

一句话：**本机做重活与多账号操作，远程机承担编译、黑苹果专属与服务器任务；git 提交两台都可进行。**

---

## 二、全维度差异对比总表

| 维度 | 本机 cw | 远程机 wj |
|---|---|---|
| **角色定位** | 全能开发+设计+日常 | 黑苹果编译/服务器（git 两机均可提交） |
| **包管理** | Homebrew（/usr/local，206 formulae+10 casks，时点值）+ mise(node/java/maven/gradle) + cargo | Homebrew（/usr/local，170 formulae+6 casks，时点值）+ 系统 gem；**非交互 SSH 的 PATH 不含 /usr/local/bin，需绝对路径** |
| **IDE** | VS Code(14 扩展) + Android Studio + IntelliJ + WebStorm + emacs + Codex | VS Code(无 CLI) + Trae + Xcode |
| **软件数量** | ~130 个应用 | ~70 个应用 |
| **GitHub 账号** | 三账号 SSH 分流（byte886/tinyverse/web3） | 单账号（主力） |
| **ssh-agent** | ✅ 3 把常驻（launchd+钥匙串） | ✅ 已修复常驻（2026-09-15，launchd+钥匙串+`.zshenv`） |
| **git credential** | 正常（主仓走 SSH，不用 helper） | gh helper 配置在、gh 已装于 /usr/local/bin；主仓走 SSH 实际不依赖 |
| **硬件** | i5-1260K / 64G / 1TB NVMe | i5-13600KF / 128G / 4TB NVMe + 16TB HDD |
| **内存压力** | 🔴 偏高（64G 易占满，以 top 实测为准） | 128G，占用以 top 实测为准（盘点时约 119G） |
| **系统版本** | macOS 15.7.8 | macOS 15.7.8（黑苹果，多系统分区） |
| **Xcode** | 26.3.0 | 16.4.0 |
| **git 用户** | softwarecheng@126.com | softwarecheng@126.com（一致） |

---

## 三、一致性维护策略

### 3.1 必须保持一致

| 项 | 原因 |
|---|---|
| SSH 私钥（3 把，已同步） | 双机互访与 GitHub 访问基础 |
| `~/Doubao` git 仓库 | 双机共享工作区，靠 git 同步 |
| 技能文件（`~/Doubao/skills/`） | 本技能及其他技能双机共用 |
| git 用户配置（user.name / user.email） | 提交署名一致 |

### 3.2 允许差异（刻意设计）

| 项 | 说明 |
|---|---|
| 包管理器规模 | 两台都装 Homebrew（均在 /usr/local）；远程机保持精简、不追包数量，新增包按角色权衡 |
| IDE | 远程机只需基础（VS Code/Trae/Xcode），重型 IDE 不装 |
| 软件 | 按角色选配：本机全能，远程机加微信多开/Warp/Devin/Macs Fan Control |
| GitHub 账号分流 | 仅本机配三账号，远程机单主力账号 |

### 3.3 不对称项现状

| 项 | 机器 | 状态 / 处理 |
|---|---|---|
| ~~ssh-agent 未运行~~ | 远程机 | ✅ 已修复（2026-09-15）：launchd agent + 钥匙串 + `.zshenv` 持久化，见 security-and-git.md 第三节 |
| ~~ed25519 / id_rsa 两机指纹不一致~~ | 两机 | ✅ 已对齐（2026-09-15 以 .8 为准；旧 `.migrated-bak` 留底已于 2026-09-16 随云主机退役一并删除），见 security-and-git.md 第二节 |
| ~~credential helper 失效~~ | 远程机 | 已随 gh 安装（/usr/local/bin/gh）自行解决；主仓走 SSH，无需再处理 |

---

## 四、双机同步 SOP（两机均可提交，对端只快进）

> 完整命令步骤只在 [sop.md 第二节](sop.md) 维护一处（权威），本节不复制，避免两处漂移。要点：当前机正常 commit/push（含子模块遵循"先子后父"）；对端 `fetch` → 确认 `rev-list origin/main..HEAD` 为空 → 仅 `merge --ff-only` → `submodule update`，**禁止 `git pull` / 合并提交**。

- 同一套 `~/Doubao` 在两台 Mac 间同步，**两台均可提交**，对端只做 ff-only；submodule / gita 的通用机制由 mac-system-toolkit 的 git 多仓能力提供，本节只管"两台机如何协作"。
- 技能与脚本内**禁止硬编码 `/Users/<用户名>`**：Shell 用 `$HOME`、Python 用 `Path.home()`、文档示例用 `~`；引号内/配置框内 `~` 不展开，用 `$HOME`。

---

## 五、新工具/新软件接入的双机决策流程

装新东西前先过这张表，决定装一台还是两台：

| 类型 | 装两台？ | 判断标准 |
|---|---|---|
| SSH 密钥、git 配置、技能文件、`~/Doubao` 内容 | ✅ 必须两台 | 属于「必须一致」项 |
| 日常通讯/办公/浏览器软件 | ✅ 建议两台 | 行为对齐，避免差异 |
| 重型 IDE / 设计工具 / 效率工具 | ❌ 仅本机 | 远程机保持精简 |
| 命令行工具 | 按需 | 两台都有 Homebrew；远程机保持精简，新增 CLI 前先确认符合其角色 |
| 黑苹果专属（散热监控、多系统、微信多开、AI 开发机任务） | ✅ 仅远程机 | 角色所需 |
| 加密货币/节点/服务类 | ❌ 默认仅本机 | 远程机只做编译提交 |

**决策口诀**：「基础与凭证两台一致；重活与工具只装本机；远程机专属（散热/多开/AI 任务）只装远程机。」
