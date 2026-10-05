# ClashX Pro 命令行控制（系统代理 + 出站模式）

> 2026-10-04 实测沉淀。来源：ClashX 官方 GitHub（README + Shortcuts.md）+ 本机实测验证。
> 场景：需要命令行开关 ClashX Pro 系统代理、切换出站模式，且要求 ClashX 菜单勾选状态同步时使用。

## 核心结论

- **ClashX Pro 官方支持 AppleScript 通道**：`osascript -e 'tell application "ClashX Pro" to <command>'`，由 ClashX 自身执行，菜单勾选同步，无需辅助功能权限。
- **官方命令只有两个**（来源 Shortcuts.md）：
  - `toggleProxy`：开/关系统代理（toggle，点一次开、再点关）
  - `proxyMode "rule"|"global"|"direct"`：切换出站模式
- **Clash 引擎 API（external controller，本机 127.0.0.1:61111）管不到系统代理**：实测 `/system-proxy` 等端点 404；`SystemProxyManager.swift` 是客户端代码。API 只负责模式/节点/连接/规则。

## 封装脚本（本机 ~/bin，已入 PATH）

| 命令 | 作用 |
|---|---|
| `proxy-on` | 打开系统代理（幂等：已开则提示不操作） |
| `proxy-off` | 关闭系统代理（幂等：已关则提示不操作） |
| `proxy-mode rule\|global\|direct` | 切换 ClashX 出站模式 |
| `proxy-status` | 查看系统代理状态 + ClashX 出站模式 |

脚本内部：先 `scutil --proxy` 判断当前状态再决定是否 `toggleProxy`，避免误切换。

## 底层命令（不依赖脚本，可直接用）

```bash
# 开关系统代理（toggle）
osascript -e 'tell application "ClashX Pro" to toggleProxy'

# 切出站模式
osascript -e 'tell application "ClashX Pro" to proxyMode "global"'   # 全局
osascript -e 'tell application "ClashX Pro" to proxyMode "direct"'  # 直连
osascript -e 'tell application "ClashX Pro" to proxyMode "rule"'    # 规则

# 查看系统代理状态
scutil --proxy
```

## Clash 引擎 API（127.0.0.1:61111）

- 端口/secret 存在偏好设置：`defaults read com.west2online.ClashXPro apiPort` / `api-secret`（**secret 属凭证，不外露明文**，用时以命令读取）
- 常用端点：`GET /version`、`GET /configs`、`GET /proxies`（节点组+当前选中）、`GET /connections`、`PATCH /configs`（切 mode）、`PUT /proxies/<组>`（切节点）
- **不能**：开关系统代理（无此端点）、TUN 等客户端级功能

```bash
SECRET=$(defaults read com.west2online.ClashXPro api-secret)
curl -H "Authorization: Bearer $SECRET" http://127.0.0.1:61111/configs
curl -X PATCH -H "Authorization: Bearer $SECRET" -d '{"mode":"global"}' http://127.0.0.1:61111/configs
```

## 已排除的方案（不必再试）

- `networksetup -setwebproxy ...`：能开系统代理但**绕过了 ClashX**，菜单勾选不同步，且需逐项设置 HTTP/HTTPS/SOCKS/bypass，弃用
- System Events 点击菜单（AppleScript UI 脚本）：可行但需要辅助功能权限、路径脆弱，官方 toggleProxy 更优
- Clash API 开系统代理：无端点（404），不存在

## 参考

- 官方 README：https://github.com/ClashX-Pro/ClashX（Global Shortcuts 一节明确 "Or use AppleScript - see Shortcuts Guide"）
- 官方 Shortcuts.md：https://github.com/ClashX-Pro/ClashX/blob/master/Shortcuts.md
- 本机 ClashX Pro 版本：1.118.1.1（apiPort=61111，mixed-port=7890，mode=rule）
