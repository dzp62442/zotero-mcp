# Codex 支持开发进度

更新时间：2026-03-05

## 0. 当前里程碑

| Phase | 目标 | 状态 | 说明 |
|---|---|---|---|
| Phase 1 | 协议兼容修复（握手） | 已完成 | `notifications/initialized` 与 notification 语义已上线 |
| Phase 2 | Codex 配置生成与 UI | 已完成 | TOML + headers + UI 顺序 + i18n 已落地 |
| Phase 3 | 文档一致性修复 | 已完成 | README/README-zh 已按现实现修订 |
| Phase 4 | 本地回归与发布准备 | 进行中 | 当前仅做本地验证，不发布不推送 |

## 1. 已完成提交

1. `50477bc` `docs: add codex support review and execution plan`
2. `50e2811` `fix: support codex initialized notification handshake`
3. `908c016` `docs: update codex progress and phase1 validation status`
4. `b8d879f` `feat: add codex client config and align invalid request status`
5. `09a55fd` `fix: align codex order and header compatibility with claude config`

## 2. Phase 1 结果（协议层）

实现：

1. 支持 `notifications/initialized` 与 legacy `initialized`。
2. notification 返回 `202 + 空 body`。
3. batch 请求返回 `-32600`。
4. `-32600/-32700` 统一映射为 HTTP `400`。
5. `supportedMethods` 包含 `notifications/initialized`。

运行态检查（2026-03-05）：

1. `GET /mcp` -> `200`
2. `POST initialize` -> `200`
3. `POST notifications/initialized` -> `202` + `Content-Length: 0`
4. `POST tools/list` -> `200`
5. `POST` batch -> `400 + -32600`

## 3. Phase 2 结果（配置/UI）

实现：

1. 新增 Codex 客户端配置生成。
2. 新增 TOML 渲染分支。
3. Codex TOML 默认带 headers：
- `[mcp_servers."zotero-mcp".headers]`
- `"Content-Type" = "application/json"`
4. 偏好设置客户端顺序：
- `Claude Code` 第一
- `Codex CLI` 第二
5. 中英文指引均包含“与 Claude Code / cc-switch 统一配置”。

## 4. Phase 3 结果（文档）

本地已完成：

1. `README.md` 更新为插件内置 HTTP 架构。
2. `README-zh.md` 更新为同口径内容。
3. 新增 Codex CLI 接入与 TOML 示例。
4. 明确握手链路与故障排查。
5. 清理旧链路与旧端口描述。

## 5. 仍需本地执行项（Phase 4）

1. 在 `zotero-mcp-plugin` 执行 `npm run build`。
2. 安装本地构建包 `.scaffold/build/zotero-mcp-plugin.xpi`。
3. 用 Codex CLI 执行一次真实工具调用回归。
4. 用 Claude Code 回归一次统一 headers 配置。
5. 汇总本地回归结果后再评估是否进入发布流程。

## 6. 风险与注意事项

1. Codex CLI 上游配置语法可能变化，发布前需再核一次。
2. 本地代理/VPN 可能影响 `127.0.0.1` 访问。
3. 当前策略不支持 batch 请求，文档与实现必须持续一致。
