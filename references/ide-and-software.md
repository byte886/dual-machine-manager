# IDE 与常用软件盘点

> IDE 与常用软件的双机对比；应用清单为时点快照，以 `/Applications` 现场为准。

---

## 一、IDE / 开发工具对比总表

| 工具 | 本机 cw | 远程机 wj |
|---|---|---|
| **VS Code** | ✅ code CLI `/usr/local/bin/code`，扩展约 14 个（时点） | ✅ `.app` + code CLI 软链 `/usr/local/bin/code`（已建） |
| **Android Studio** | ✅ | ❌ |
| **IntelliJ IDEA** | ✅ | ❌ |
| **WebStorm** | ✅ | ❌ |
| **TRAE SOLO CN** | ✅ | ✅（另有 `Trae CN.app`） |
| **Codex.app** | ✅ | ❌ |
| **Xcode** | ✅ Xcode-26.3.0 | ✅ Xcode-16.4.0 |
| **emacs** | ✅ `/usr/local/bin/emacs`（brew） | ❌ |
| **vim** | ✅ `/usr/bin/vim` | ✅ `/usr/bin/vim` |
| **neovim** | ❌ | ❌ |
| **Cursor / Windsurf / Zed / Sublime** | ❌ 均未装 | ❌ 均未装 |

---

## 二、本机 VS Code 扩展清单（14 个）

| 扩展 ID | 分类 |
|---|---|
| `rust-lang.rust-analyzer` | Rust |
| `1yib.rust-bundle` | Rust |
| `mooman219.rust-assist` | Rust |
| `golang.go` | Go |
| `r3inbowari.gomodexplorer` | Go |
| `trixnz.go-to-method` | Go |
| `yzhang.markdown-all-in-one` | Markdown |
| `bierner.markdown-preview-github-styles` | Markdown |
| `cweijan.vscode-office` | Office 文档查看 |
| `grapecity.gc-excelviewer` | Office 文档查看 |
| `alefragnani.project-manager` | 项目管理 |
| `jamesmaj.easy-icons` | 图标/UI |
| `ms-ceintl.vscode-language-pack-zh-hans` | 中文语言包 |

> 扩展侧重：**Rust（3）、Go（3，含 go-to-method）、Markdown（2）、Office 文档查看（2）**。远程机已有 code CLI，用 `ssh wj '/usr/local/bin/code --list-extensions'` 采集（非交互 PATH 不含 /usr/local/bin，用绝对路径）。

---

## 三、远程机 code CLI（已具备）

远程机 `/usr/local/bin/code` 软链已存在（指向 VS Code.app 内 `Contents/Resources/app/bin/code`），终端可直接用。若将来在新机上缺失，两种建法：

```bash
# 界面：Cmd+Shift+P → "Shell Command: Install 'code' command in PATH"
# 命令行：sudo ln -s "/Applications/Visual Studio Code.app/Contents/Resources/app/bin/code" /usr/local/bin/code
```

采集远程扩展（注意非交互 PATH，用绝对路径）：
```bash
ssh wj '/usr/local/bin/code --list-extensions'
```

---

## 四、常用软件对比（/Applications）

### 4.1 两台共有

| 分类 | 软件 |
|---|---|
| 浏览器 | Google Chrome, Safari, 豆包浏览器 |
| 通讯 | WeChat, 企业微信, QQ, Telegram, Lark(飞书), TencentMeeting |
| 办公 | Microsoft Word / Excel / PowerPoint, wpsoffice, OneDrive |
| 开发工具 | iTerm, Postman, Navicat Premium, Commander One, RDM(Redis), QtScrcpy |
| 系统工具 | Alfred 5, Spectacle, Keka, CheatSheet, Tuxera Disk Manager, OCLP-Mod, OpenCore Configurator, Blackmagic Disk Speed Test |
| 远程/网络 | RustDesk, Tailscale, BaiduNetdisk_mac |
| 其他 | IINA, ACE Studio, 元宝, 抖音, 汽水音乐, Doubao, WorkBuddy |

### 4.2 仅本机 cw 有

| 分类 | 软件 |
|---|---|
| 浏览器 | Microsoft Edge |
| 通讯 | Discord, Microsoft Teams classic |
| 办公 | Microsoft OneNote, Notion, Obsidian, myBase |
| IDE | Android Studio, IntelliJ IDEA, WebStorm, Codex, emacs |
| 设计/媒体 | CapCut, HandBrake, Acorn, Canva, draw.io, ScreenFlow, Wondershare Filmora, Free Ruler, CamTwist |
| 开发工具 | kitty, DataGrip, Proxyman, Hex Fiend, PlistEdit Pro, Reactotron, wechatwebdevtools, Cocos, Bitcoin-Qt, IPFS Desktop, Ollama, CodeSwitch, App Cleaner 7 |
| 系统/效率 | Raycast, Moom, Keyboard Maestro, BetterZip, CleanMyMac_5, Cocktail, MonitorControl, iShot, AutoSwitchInput, Input Source Pro, Eye Monitor, XtraFinder |
| 网络/安全 | Clash Verge, BitBrowser Global, Chrome Remote Desktop |
| 其他 | Anki, 影刀, 扣子, 夸克网盘, 亿图图示, 大黄蜂云课堂, Claude, ChatWise, hisuite, Intel Power Gadget |

### 4.3 仅远程机 wj 有

| 分类 | 软件 | 备注 |
|---|---|---|
| 浏览器 | Chrome Gemini, Gemini 2, UC | — |
| 通讯 | WeChat_backup_37335.app, WeChat_tampered_37342.app | **历史多开试验遗留（已废弃，见下）** |
| 办公 | Microsoft Outlook, Microsoft OneNote, LibreOffice, PDF Expert, PDF Professional Suite, Foxit Phantom | PDF 工具集中 |
| 设计/媒体 | GIMP, Aerial（屏保）, res-downloader | — |
| 开发工具 | Warp, QClaw, Devin, DoubaoWork, LANDrop | Warp/Devin 为本机无 |
| 系统工具 | CleanMyMac X, **Macs Fan Control**, Xnip, SogouInputSwitchHelper, lghub, REALFORCE Connect | Macs Fan Control = 黑苹果散热监控 |
| 远程/网络 | ToDesk, UURemote, ClashX Pro, Proxifier | 远程机自带远程桌面栈 |

> 📌 **特别注意**：
> - **微信多开（WeChat_backup / WeChat_tampered）是历史试验遗留，已验证会触发风控封号，方案废弃、不要再复活、不要当能力使用**；当前统一为单官方 App 内切换账号（见 wechat-control 技能 new-account-sop §A.4.2）。这两个 .app 在远程机上，是否删除由用户决定，不擅删。
> - Warp、Devin（AI 开发任务）、Macs Fan Control（黑苹果散热监控）才是远程机的角色特征，保留。

---

## 五、差异总结与一致性维护建议

### 差异总结
- 本机是「全能工作站」：IDE 全家桶（Android Studio/IntelliJ/WebStorm/Codex）+ 设计/效率工具齐全。
- 远程机是「黑苹果编译/服务器」：VS Code（含 CLI）+ Trae + Xcode，另有 Warp、Devin、PDF 工具和 Macs Fan Control；微信多开为已废弃遗留。

### 维护建议
1. **IDE 不追求对称**：远程机不必装 IntelliJ/WebStorm 等重型 IDE，远程编辑走 SSH + 本机 VS Code Remote 即可。
2. 远程机 code CLI 已具备，用绝对路径 `ssh wj '/usr/local/bin/code --list-extensions'` 即可采集/对比扩展。
3. **远程机角色软件保留**：Macs Fan Control（黑苹果散热）、Devin/Warp（AI 开发）；微信多开遗留已废弃、勿当能力使用，删除与否由用户定。
4. **共有软件优先双机版本一致**：Chrome/飞书/IINA 等日常软件两端大版本尽量对齐，避免行为差异。
