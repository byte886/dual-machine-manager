# 远程访问档案（RustDesk + Tailscale）

> 记录两台机器之间的远程访问方式：RustDesk（ID/版本/服务/密码状态/连接路径）与 Tailscale（节点对照）。易变项（在线状态、版本、ID）以现场实测为准；本文档 2026-10-05 由 wj 侧实测录入。

## 一、总览（2026-10-05 实测）

| 项目 | cw（192.168.2.16） | wj（192.168.2.15，本机） |
|---|---|---|
| RustDesk ID | **180 767 432**（实测） | **120 029 537** |
| RustDesk 版本 | 1.4.9 | 1.4.9 |
| RustDesk 服务 | 开（`service` + `RustDesk --server`） | 开（`service` + `RustDesk --server` 两个进程） |
| RustDesk 密码 | 已配置（加密存储，值不入库） | 已配置（加密存储，值不入库） |
| Tailscale 节点名 | `mac-jack-dev-1` | `mac-jack-dev-2` |
| Tailscale IP | 100.83.238.64 | 100.109.91.22 |
| 局域网 IP（静态） | 192.168.2.16 | 192.168.2.15 |
| 2026-10-05 在线状态 | **在线**（复测通过） | 在线 |

## 二、RustDesk

- **ID 来源**：wj 的 `120 029 537` 与 cw 的 `180 767 432` 均为 2026-10-05 各机 `RustDesk --get-id` 实测。wj 的 `RustDesk_lan_peers.toml` 中另有旧记录 `472 400 819`（username=chenwenjie、hostname=mac-pro.local），为**历史残留 ID**（重装/重置后已变），以实测为准。
- **服务器**：官方公共服务器 `rs-ny.rustdesk.com:21116`（见 wj 的 RustDesk2.toml），未自建服务器。
- **连接路径（由近到远）**：同局域网直连 → 同 tailnet 经 Tailscale → 官方中转兜底。
- **配置落点（macOS）**：`~/Library/Preferences/com.carriez.RustDesk/`（RustDesk.toml / RustDesk2.toml / RustDesk_lan_peers.toml / RustDesk_ab 联系人 / RustDesk_group）。
- **手机端**：Android 已装 RustDesk 1.5.0（2026-10-05 经 adb 安装，arm64）。控制两台主机只需各自 ID + 密码；**建议两台主机均设固定密码**（RustDesk 设置 → 安全），否则临时密码会变。
- **版本差异**：主机 1.4.9 / 手机 1.5.0，官方客户端跨小版本一般可互连；如连接异常，先对齐版本排查。

## 三、Tailscale

- **节点对照**（2026-10-05 `tailscale status` 实测）：

| 节点 | Tailscale IP | 对应机器 | 状态 |
|---|---|---|---|
| mac-jack-dev-1 | 100.83.238.64 | cw | online（2026-10-05 复测） |
| mac-jack-dev-2 | 100.109.91.22 | wj（本机） | online |
| mac-jack-dev-3 | 100.92.65.17 | **未知 / 疑似旧设备** | offline（last seen 3d，归属待确认） |

- 使用规则与历史代理冲突教训见 machines.md「Tailscale 与系统代理」节；开启 Tailscale 后断网排障见 sop.md 6.4——不在此重复。

## 四、互联现状与注意事项（2026-10-05 实测）

- **复测结论（2026-10-05，cw 开机后）**：✅ **网络层互联验证通过**——LAN ping 0% 丢包（约 1.2ms）、SSH 通、cw 的 RustDesk 直连端口 21118 开放、Tailscale 双节点在线；两端 RustDesk 服务运行、版本一致（1.4.9）、同一官方服务器（rs-ny.rustdesk.com:21116）。
- **历史记录**：wj 日志显示 2026-10-04 曾向旧 ID `472400819` 发起会话（TCP 建连 624µs），但当日 DNS 异常导致直连 `100.83.238.64:21118` 失败——为当时的网络/DNS 问题，已随网络恢复正常。
- **剩余验证**：手机端实际连接测试（cw 用新 ID `180 767 432`）待执行。
- **排查要点**：cw 关机时手机无法连接（非无人值守常开）；建议两台主机确认**开机自启**（Windows/macOS 装为服务或登录项）并设固定密码，手机才能随时连接。

## 五、复测清单（cw 开机后执行）

1. ✅ `ping 192.168.2.16` / `ssh cw` 确认可达（0% 丢包，2026-10-05）
2. ✅ cw 侧 RustDesk：服务运行、版本 1.4.9、ID 实测为 **180 767 432**（旧记录 472 400 819 已废弃）、密码已配置
3. ✅ 网络层连接验证：LAN ping / SSH / 直连端口 21118 / Tailscale 双在线
4. ⏳ 手机 → 两台主机实际连接测试（待执行）
5. ✅ 复测结果已回填本文档「总览」与「互联现状」

## 六、cw 无显示器无法进系统（黑苹果排障，2026-10-05）

- **现象**：cw 不接显示器时，大部分情况下卡黑屏进不了 macOS；接显示器正常。
- **cw 实测 boot-args**（nvram）：`alcid=66 -wegnoigpu ipc_control_port_options=0`——**iGPU（UHD 770）已禁用**，显示设备只剩 RX 460/560（Polaris）。
- **判断**：iGPU 禁用 + Polaris 独显在无显示器时拿不到 EDID，macOS 显卡驱动初始化挂起 → 黑屏。
- **方案（按优先级）**：
  1. **HDMI/DP 假负载（推荐）**：插在显卡输出口，让显卡认为有显示器，无人值守启动稳定通过
  2. 试加 boot-args `agdpmod=vit9696`（绕过显卡 board-id 检查，对 Polaris 黑屏有效）
  3. BIOS 确认：CSM 关闭、Above 4G Decoding 开启、主显示设备设 PCIe（独显）
- **影响**：不解决则 cw 重启后手机无法远程连接（系统起不来，RustDesk 不运行）；解决后 cw 方可无人值守。
