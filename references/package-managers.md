# 包管理工具深度盘点

> 本机 `cw` = chenwenjie @ 192.168.2.8；远程机 `wj` = wenjiechen @ 192.168.2.9（黑苹果）。
> 两台都装 Homebrew（prefix 均为 /usr/local）；版本/包数量是易变时点值，以现场 `brew list` / `<cmd> --version` 实测为准。
> **关键坑**：远程机非交互 SSH 的 PATH 不含 /usr/local/bin，`ssh wj brew` 会误报 command not found——用 `/usr/local/bin/brew` 或先 `export PATH=/usr/local/bin:$PATH`，并非没装。

---

## 一、全量对比总表

| 包管理器 | 本机 cw | 远程机 wj | 差异要点 |
|---|---|---|---|
| **Homebrew** | ✅ prefix `/usr/local`，206 formulae + 10 casks（时点值） | ✅ prefix `/usr/local`，170 formulae + 6 casks（时点值），非交互 PATH 不含 /usr/local/bin | 两台都有，本机更全 |
| **npm/node** | 系统级由 mise 提供（Doubao sandbox 内另带 npm） | ✅ brew node/npm 在 /usr/local/bin（版本实测为准） | 来源不同，远程补 PATH 即可用 |
| **pip3/python3** | brew python@3.10~3.14（sandbox 内另带 pip） | brew python@3.12/3.14（/usr/local/bin 优先）；/usr/bin 另有 Xcode 旧 3.9 | 远程补 PATH 后即为新版 |
| **gem** | ✅ 3.6.3，`/usr/local/opt/ruby/bin/gem`，89 个 gem（多为 ruby 默认自带） | ✅ 3.0.3.1（系统自带） | 远程 gem 版本旧 |
| **cargo / rust** | ✅ rustup 多 toolchain（另含 brew rust），版本实测为准 | ✅ /usr/local/bin brew rust 1.95，另可能有 rustup | 固定版本用 rust-toolchain.toml |
| **mise**（版本管理） | ✅ `~/.local/bin/mise`，管 node/java/maven/gradle | ❌ 无 | 仅本机 |
| **nvm** | ❌ 无（已改用 mise） | ❌ 无 | — |
| **pnpm** | ✅ mise shim | ❌ 无 | 仅本机 |
| **yarn** | ✅ `/usr/local/bin/yarn`（brew） | ❌ 无 | 仅本机 |
| **go** | ✅ `~/go/bin/go` | ✅ `/usr/local/bin/go`（brew） | 两台都有 |
| **java** | ✅ mise temurin-17 | ⚠️ 仅 /usr/bin/java 桩、无 JRE（报 Unable to locate），需要时 brew install | 远程暂无可用 JDK |
| **maven** | ✅ mise 3.9.16 | ❌ 无 | 仅本机 |
| **gradle** | ✅ mise 8.1.1 | ❌ 无 | 仅本机 |
| **gita**（多 git 仓总览/批量） | ✅ `/usr/local/bin/gita`（brew），`gita ll` 已纳管仓库群 | ✅ `/usr/local/bin/gita`（brew），已纳管 | 双机均装；**用法归 mac-system-toolkit 的 git 多仓能力，本表只登台账**；纳管清单 `~/.config/gita/repos.csv` 机器相关不入库 |

---

## 二、各包管理器详情

### 2.1 Homebrew（双机均装，本机更全）

- **路径**：两台 prefix 都是 `/usr/local`（Intel Mac 默认路径），均为手动安装、非系统自带。
- **规模（时点值，以 `brew list` 为准）**：本机约 206 formulae + 10 casks；远程机约 170 formulae + 6 casks。
- **远程机 PATH 坑**：非交互 ssh 默认 PATH 不含 /usr/local/bin，远程调用一律用 `/usr/local/bin/brew` 或命令前 `export PATH=/usr/local/bin:$PATH`。
- **用途定位**：两台命令行工具链与 GUI 应用的统一入口。

#### formulae 分类摘要（本机，时点快照；以 brew list 为准）

| 分类 | 关键包（节选） |
|---|---|
| **开发工具链** | cmake, gcc, llvm, ninja, meson, make, autoconf, automake, pkg-config, swig, lld, tree-sitter, emacs, ripgrep (rg), fzf, jq, wget, curl, netcat, nmap, tmux, tmuxinator, screen, watch, tree, hugo |
| **运行时/语言** | python@3.10~3.14（5 个版本）, ruby, lua, openjdk, pyenv, uv, yarn |
| **网络/服务** | nginx, postgresql@14, mysql, docker, docker-completion, autossh, tailscale 相关 |
| **媒体处理** | ffmpeg 全套（x264/x265/svt-av1/dav1d/libvpx/libvmaf/opus/lame/webp）, HandBrake 依赖 |
| **加密货币相关** | bitcoin, bfgminer, electrum, ord, parity, sui, rocksdb, leveldb, zeromq, libsodium, libusb |
| **其他** | gh, gita, tldr, nexttrace, displayplacer, trzsz-ssh, trzsz-go, miniupnpc, gnutls |

#### casks（10 个）

`cc-switch`, `chromedriver`, `electrum`, `font-fira-code`, `keycastr`, `rustdesk`, `stretchly`, `tailscale-app`, `temurin`, `wechattweak-cli`

#### Homebrew on Intel 支持时间线与替代预案（2026-09 查官方）

两台均为 Intel 黑苹果，Homebrew 对 Intel(x86_64) 的退场是确定时间线（来源见末尾官方链接）：

- **2026-09（Homebrew 7.0 起）**：Intel x86_64 降为 **Tier 3**——停止为 Intel 构建新 bottle（二进制包）、撤 CI；**但已装的 Homebrew 仍可正常运行**，存量包照用，只是装新包可能无预编译件而回退源码编译（黑苹果上慢且易失败）。
- **2027-09 起**：**彻底移除**在 Intel 上运行 Homebrew 的能力。
- 背景：macOS Tahoe 26 是最后一代支持 Intel 的系统，macOS 27 弃 Intel；GitHub Actions 于 2027 秋退役 Intel macOS runner。官方对 Intel 的点名替代是 **MacPorts**（仍持续支持 Intel macOS），`brew doctor` 在无 bottle 时也会建议 MacPorts。

对本双机的应对（**不急于迁移，有约一年缓冲，以"用到装不上"为触发点**）：

1. **语言运行时不依赖 brew bottle**：node/java/maven/gradle 统一走 mise（见 2.6），brew 退场不影响开发运行时；
2. **GUI 应用走 cask**：cask 装的是上游 `.app`，基本不受 core formula 停建 bottle 影响；
3. **命令行工具**：rg/fd/jq/ffmpeg/gh/gita 等存量包一直可用；2027 前若新工具在 brew 无 Intel bottle，再按需评估 MacPorts、官方二进制或源码，不提前整体迁移。

> 来源：Homebrew 官方 Support Tiers（https://docs.brew.sh/Support-Tiers）、Homebrew 7.0.0 release notes（https://brew.sh/2026/09/13/homebrew-7.0.0/）。

### 2.2 npm（sandbox 隔离，非系统级）

- **现状**：本机 npm 10.9.8 位于 Doubao sandbox 运行时内（`.../sandbox_runtime/bases/.../bin/npm`），**系统级 shell 中没有 npm**。
- **全局包**：仅 corepack + npm 两个。
- **含义**：需要在终端全局安装 node CLI 工具时，不能依赖 `npm i -g`；node 运行时与 pnpm 实际由 **mise** 提供（见 2.6）。
- **远程机**：brew 已装 node/npm（/usr/local/bin）；非交互 SSH 下同样要补 PATH。

### 2.3 pip3 / Python

- **本机**：brew 安装 python@3.10~3.14 多版本；sandbox 内另带 pip。
- **远程机**：/usr/local/bin 有 brew python@3.12/3.14（补 PATH 后优先、较新）；/usr/bin/python3 仍是 Xcode 自带 3.9（旧，勿误用）。

### 2.4 gem

- **本机**：3.6.3，`/usr/local/opt/ruby/bin/gem`（brew ruby），89 个 gem，绝大多数为 ruby 默认自带。
- **远程机**：3.0.3.1，系统自带。

### 2.5 cargo / rustup（两台都有）

| 项目 | 本机 cw | 远程机 wj |
|---|---|---|
| 来源 | rustup 多 toolchain（另含 brew rust） | /usr/local/bin brew rust 1.95，另可能有 rustup |
| 版本 | 以 `cargo --version` / `rustup toolchain list` 实测为准 | 同左 |

两台都有 Rust 工具链；跨机编译注意版本差异，固定版本的项目用 `rust-toolchain.toml` 锁定，不依赖某台机器的默认版本。

### 2.6 mise（本机版本管理器）

- **路径**：`~/.local/bin/mise`
- **管理对象**：
  - node 23.11.1（pnpm 为其 shim）
  - java temurin-17.0.20
  - maven 3.9.16
  - gradle 8.1.1
- **作用**：替代 nvm/jenv 等单一语言管理器，统一管理 node/java/maven/gradle 版本，是本机「无系统级 npm」问题的实际解法。
- **远程机**：未安装。

---

## 三、远程机 Homebrew 现状与非交互 PATH 应对

### 现状
远程机**已装** Homebrew（/usr/local，约 170 formulae），node/npm、python3.14、go、rust、aria2、autoconf、7z 等常见工具齐全，并非"无包管理"。

### 唯一高频坑：非交互 SSH 的 PATH
`ssh wj '<cmd>'` 的默认 PATH 是 `~/.cargo/bin:/usr/bin:/bin:/usr/sbin:/sbin`，不含 /usr/local/bin，于是 brew 装的工具会 command not found（看着像没装）。三种解法：

```bash
ssh wj '/usr/local/bin/brew list'                       # 1) 直接绝对路径
ssh wj 'export PATH=/usr/local/bin:$PATH; brew list'    # 2) 命令前补 PATH
ssh wj -t 'zsh -lc "brew list"'                         # 3) 走登录 shell（-t + -l）
```

### 新增工具原则
远程机定位精简，新增 CLI 前先确认符合其角色；能在本机处理的重型工具链不必两边都装。

---

## 四、包管理一致性维护建议

1. **远程机已装 Homebrew 但保持精简**，新增包前确认符合其角色；非交互 SSH 注意用绝对路径或补 PATH。
2. **本机新工具优先走 Homebrew**（`brew install`），装完在 `references/package-managers.md` 记录关键包。
3. **node/java 相关以 mise 为准**，不再引入 nvm/jenv，避免多套版本管理器并存。
4. **cargo toolchain 保持双机兼容**：跨机编译的 Rust 项目，默认 toolchain 版本差距较大（1.81 vs 1.70），涉及固定版本的项目用 `rust-toolchain.toml` 锁定，不依赖机器默认。
5. **Python 注意解释器来源**：远程机 /usr/bin/python3 是旧 3.9，新语法脚本用 /usr/local/bin/python3（3.12/3.14）或在脚本头写死解释器路径，避免落到系统旧版。
