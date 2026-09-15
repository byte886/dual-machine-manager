# 机器健康度

> 本文记录资源占用的判断方法、磁盘/进程问题与只读巡检命令。CPU/内存/容量都是易变时点值，以第八节命令现场实测为准。

---

## 一、资源占用对比

| 指标 | 本机 cw | 远程机 wj | 现场查看 |
|---|---|---|---|
| CPU | i5-12600K（10 核/16 线程） | i5-13600KF（14 核/20 线程） | `top -l1 -n0` |
| 内存总量 | 64 GB | 128 GB | `sysctl hw.memsize` |
| 内存占用 | 🔴 长期偏高、易占满（盘点时仅剩百余 MB） | 盘点时约 119G/128G，以实测为准 | `top -l1 -n0 \| grep PhysMem` |
| Swap | 无明显颠簸 | 无明显颠簸 | `vm_stat` |
| 电源 | AC（台式机，无电池） | AC（黑苹果台式机，无电池） | — |
| 温度/风扇 | powermetrics 无 SMC 读数 | powermetrics 无 SMC 读数（黑苹果，用 Macs Fan Control） | — |

---

## 二、🔴 需优先处理（4 项）

| # | 问题 | 机器 | 说明 | 修复建议 |
|---|---|---|---|---|
| 1 | **内存长期占满** | 本机 cw | 64G 物理内存常被吃满（实测仅剩百余 MB），Parallels/IDE 全家桶是大户 | `top -o mem` 排序定位；关闭闲置 VM/IDE，必要时重启 |
| 2 | **远程机 ssh-agent 未运行** | 远程机 wj | `SSH_AUTH_SOCK` 为空，密钥未加载 | `eval $(ssh-agent) && ssh-add --apple-use-keychain ~/.ssh/id_*` 并配 launchd 自启（详见 security-and-git.md 第三节） |
| 3 | ~~远程机 credential helper 失效~~ | 远程机 wj | **已解决**：gh 已装于 /usr/local/bin、helper 有效；主仓走 SSH 本就不依赖 | 无需处理，留档 |
| 4 | **iOS 模拟器卷 98% 满（仍存在）** | 本机 cw | CoreSimulator 约 22G 卷实测仅剩约 551M | `xcrun simctl delete unavailable` 清理旧模拟器；必要时 `xcrun simctl purge -s all` |

---

## 三、🟡 建议关注

| # | 问题 | 机器 | 说明 |
|---|---|---|---|
| 5 | 备份盘偏满 | 本机 cw | 容量以 `df -h` 实测，超 80% 关注 |
| 6 | /Volumes/sys（Windows 分区）偏满 | 远程机 wj | 多系统分区，容量以 `df -h` 实测 |
| 7 | ~/Doubao 体积大 | 远程机 wj | 远程 Home 最大户，`du -sh ~/*` 看是否缓存/历史会话可清 |
| 8 | Load Average 偏高 | 两台 | 多核下 load 高但仍有 idle 属正常，结合 idle% 判断 |
| 9 | 两台均无 GPG 签名 | 两台 | Git commit 未做 GPG 签名（需要时再配） |
| 10 | 两台均无独立密码管理器 | 两台 | 凭据全靠 macOS 钥匙串 |
| 11 | 系统待更新 | 两台 | 以 `softwareupdate -l` 实测为准，安全更新优先 |

---

## 四、🟢 健康项

- 两台系统盘用量盘点时在 43-48%，是否充裕以 `df -h /` 实测。
- 两台均无 swap 颠簸（swapin/swapout 接近 0）。
- SSH 双机互配免密 + ControlMaster 连接复用，链路通畅。
- 远程机 16TB HDD 备份盘仅用 2%，备份空间充裕。

---

## 五、磁盘空间详情

| 卷 | 本机 cw | 远程机 wj |
|---|---|---|
| 系统数据卷 | disk2s1，容量/用量以 `df -h /` 实测 | disk2s1，同左 |
| 备份盘 | disk0s2 HFS backup（用量实测） | disk4s1 APFS backup（15T，盘点时仅用约 2%） |
| 其他卷 | iOS 模拟器卷约 22G（长期 98% 满 🔴，见问题 4）；Nix Store | /Volumes/sys（Windows）、/Volumes/s、/Volumes/Ubuntu-Serv（多系统） |

---

## 六、Home 目录大户对比

> 下列大小为盘点时点值，只用于识别"谁是大户"；当前值用 `du -sh ~/* | sort -rh | head` 实测。

**本机 cw：**

| 目录 | 大小 |
|---|---|
| ~/Parallels | 175 G |
| ~/Library | 44 G |
| ~/Desktop | 4.3 G |
| ~/go | 1.5 G |
| ~/sdk | 358 M |

**远程机 wj：**

| 目录 | 大小 |
|---|---|
| **~/Doubao** | **864 G** 🟡 |
| ~/Parallels | 381 G |
| ~/Library | 82 G |
| ~/Downloads | 75 G |
| ~/Desktop | 21 G |
| ~/go | 11 G |

> 对比要点：本机最大户是 Parallels（175G 虚拟机）；远程机最大户是 ~/Doubao（864G，异常大，需排查是否为缓存/日志/历史会话产物）。

---

## 七、系统更新状态

待更新项以 `softwareupdate -l` 现场列出为准，有安全更新时优先安装。

```bash
# 查看待更新
softwareupdate -l
```

---

## 八、健康巡检命令清单（只读，可直接执行）

```bash
# === 本机 ===
# 资源/内存
top -l 1 -n 0 | head -20
vm_stat | head -10
memory_pressure | tail -3

# 磁盘
df -h
# 各卷用量
df -h / /Volumes/* 2>/dev/null

# Home 目录大户
du -sh ~/* | sort -rh | head -10

# 高内存进程
ps -Ao rss,comm | sort -rn | head -10

# 待更新
softwareupdate -l

# iOS 模拟器占用
xcrun simctl list devices unavailable | head
```

```bash
# === 远程机（经 ssh）===
ssh wj 'top -l 1 -n 0 | head -20; echo "---"; vm_stat | head -10; echo "---"; df -h; echo "---"; du -sh ~/* | sort -rh | head -10; echo "---"; ssh-add -l 2>&1; echo "---"; git config --global --get credential.helper'
```

> 远程机命令一次性覆盖：CPU/内存、磁盘、Home 大户、ssh-agent 状态、git credential 状态——即第二节 4 个 🔴 项的快速复查。
