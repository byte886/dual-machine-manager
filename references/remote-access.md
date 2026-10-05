# 远程访问档案（RustDesk + Tailscale）

> 记录两台机器之间的远程访问方式：RustDesk（ID/版本/服务/密码状态/连接路径）与 Tailscale（节点对照）。易变项（在线状态、版本、ID）以现场实测为准；本文档 2026-10-05 由 wj 侧实测录入。

## 一、总览（2026-10-05 实测）

| 项目 | cw（192.168.2.16） | wj（192.168.2.15，本机） |
|---|---|---|
| RustDesk ID | **472 400 819** | **120 029 537** |
| RustDesk 版本 | 待 cw 开机核实 | 1.4.9 |
| RustDesk 服务 | 开（见 services.md） | 开（`service` + `RustDesk --server` 两个进程） |
| RustDesk 密码 | 待核实 | 已配置（加密存储，值不入库） |
| Tailscale 节点名 | `mac-jack-dev-1` | `mac-jack-dev-2` |
| Tailscale IP | 100.83.238.64 | 100.109.91.22 |
| 局域网 IP（静态） | 192.168.2.16 | 192.168.2.15 |
| 2026-10-05 在线状态 | **离线**（LAN ping 不通、Tailscale last seen 1d） | 在线 |

## 二、RustDesk

- **ID 来源**：wj 的 `120 029 537` 由 `RustDesk --get-id` 实测；cw 的 `472 400 819` 来自 wj 的 `~/Library/Preferences/com.carriez.RustDesk/RustDesk_lan_peers.toml` 局域网对端记录（username=chenwenjie、hostname=mac-pro.local、platform=Mac OS），**待 cw 开机后在其本机复核**。
- **服务器**：官方公共服务器 `rs-ny.rustdesk.com:21116`（见 wj 的 RustDesk2.toml），未自建服务器。
- **连接路径（由近到远）**：同局域网直连 → 同 tailnet 经 Tailscale → 官方中转兜底。
- **配置落点（macOS）**：`~/Library/Preferences/com.carriez.RustDesk/`（RustDesk.toml / RustDesk2.toml / RustDesk_lan_peers.toml / RustDesk_ab 联系人 / RustDesk_group）。
- **手机端**：Android 已装 RustDesk 1.5.0（2026-10-05 经 adb 安装，arm64）。控制两台主机只需各自 ID + 密码；**建议两台主机均设固定密码**（RustDesk 设置 → 安全），否则临时密码会变。
- **版本差异**：主机 1.4.9 / 手机 1.5.0，官方客户端跨小版本一般可互连；如连接异常，先对齐版本排查。

## 三、Tailscale

- **节点对照**（2026-10-05 `tailscale status` 实测）：

| 节点 | Tailscale IP | 对应机器 | 状态 |
|---|---|---|---|
| mac-jack-dev-1 | 100.83.238.64 | cw | offline（last seen 1d） |
| mac-jack-dev-2 | 100.109.91.22 | wj（本机） | online |
| mac-jack-dev-3 | 100.92.65.17 | **未知 / 疑似旧设备** | offline（last seen 3d，归属待确认） |

- 使用规则与历史代理冲突教训见 machines.md「Tailscale 与系统代理」节；开启 Tailscale 后断网排障见 sop.md 6.4——不在此重复。

## 四、互联现状与注意事项（2026-10-05 实测）

- **现状**：两机同网段（192.168.2.x）、同 tailnet，具备局域网直连与 Tailscale 通道条件；但 **cw 当前关机/休眠，实时互联待复测**。
- **历史记录**：wj 日志（RustDesk_rCURRENT.log）显示 2026-10-04 曾向 `472400819` 发起会话（TCP 建连 624µs，LAN 级延迟），但当日 DNS 异常（admin.rustdesk.com 解析失败）导致直连 `100.83.238.64:21118` 失败——**历史互联曾受 DNS/网络影响，需以复测为准**。
- **排查要点**：cw 关机时手机无法连接（非无人值守常开）；建议两台主机确认**开机自启**（Windows/macOS 装为服务或登录项）并设固定密码，手机才能随时连接。

## 五、复测清单（cw 开机后执行）

1. `ping 192.168.2.16` / `ssh cw` 确认可达
2. cw 侧：确认 RustDesk 服务运行、版本、ID 复核（是否 472 400 819）、密码设置情况
3. wj → cw：RustDesk 实际连接测试（LAN 直连 / Tailscale / 官方中转）
4. 手机 → 两台主机连接测试
5. 复测后将结果更新到本文档「总览」与「互联现状」
