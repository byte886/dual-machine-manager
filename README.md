# dual-machine-manager

两台 Mac（本机 cw/192.168.2.8 + 远程黑苹果 wj/192.168.2.9）的统一运维管家：系统与健康巡检、SSH 互访、launchd/brew/cron 后台服务、凭证管理、包管理器与 IDE 盘点、双机 git 同步 SOP、远程关机与故障排查。

## 使用

这是豆包（及兼容 Agent）的**本地技能（Skill）**。完整能力、触发场景与操作流程见入口文档 **[`SKILL.md`](SKILL.md)**，Agent 命中时首先读取它；下列子目录按需加载，不必一次全读。

## 目录

- `SKILL.md`：技能入口、双机速览与路由
- `references/`：按需细读的档案与 SOP（机器档案、后台服务、凭证位置、包管理器、IDE 软件、安全与 Git、健康度、双机差异对比、同步 SOP 等）

## 许可

[MIT](LICENSE) © 2026 byte886
