# Zotero MCP 支持 Codex 开发方案（执行版）

更新时间：2026-03-05

## 0. 当前状态快照

1. Phase 1（协议兼容）已完成并提交。
2. Phase 2（Codex 配置生成 + UI + i18n）已完成并提交。
3. Phase 3（README/README-zh 一致性）已完成（当前文档修订）。
4. Phase 4（发布与回归）未开始，仅保留本地验证与回归准备。

相关代码提交：

1. `50e2811` `fix: support codex initialized notification handshake`
2. `b8d879f` `feat: add codex client config and align invalid request status`
3. `09a55fd` `fix: align codex order and header compatibility with claude config`

## 1. 目标与范围

目标：在不破坏现有 Claude/Cursor/Gemini/Qwen 客户端兼容性的前提下，让 `zotero-mcp` 对 Codex CLI 提供稳定可复现的 Streamable HTTP 接入能力。

范围边界：

1. 本轮硬目标为 Codex CLI。
2. Codex Desktop 若无稳定接入方式，标注“待验证”，不阻塞当前工作。
3. 只做协议层、配置层、文档层的必要改动，不做架构重写。

## 2. 关键决策

1. 兼容两条 initialized 路径：`initialized`（历史兼容）与 `notifications/initialized`（规范路径）。
2. notification 固定语义：HTTP `202` + 空 body。
3. 非法请求语义统一：`-32600` / `-32700` 对应 HTTP `400`。
4. batch JSON-RPC 数组请求明确不支持：返回 `-32600`。
5. Codex 配置默认带 `Content-Type` header，以兼容 Claude Code / cc-switch 的统一配置策略。

## 3. Phase 1：协议兼容修复（已完成）

实现点：

1. `streamableMCPServer.ts` 支持 `notifications/initialized`。
2. `MCPRequest.id` 设为可选，区分 request 与 notification。
3. notification 请求不再构造 JSON-RPC 响应。
4. `supportedMethods` 补充 `notifications/initialized`。
5. batch 请求返回 `-32600`。

运行态验收（2026-03-05）：

1. `GET /mcp` -> `200`
2. `initialize` -> `200`
3. `notifications/initialized` -> `202` + 空 body
4. `tools/list` -> `200`
5. batch 请求 -> `400 + -32600`

## 4. Phase 2：Codex 配置生成与 UI（已完成）

实现点：

1. `clientConfigGenerator.ts` 新增 `codex` 客户端。
2. 增加 TOML 输出能力（`[mcp_servers."..."]`）。
3. Codex TOML 默认包含：
- `[mcp_servers."...".headers]`
- `"Content-Type" = "application/json"`
4. 设置页客户端顺序调整为：
- 第 1 项：`Claude Code`
- 第 2 项：`Codex CLI`
5. 中英文文案补齐（含“与 Claude Code / cc-switch 统一配置”说明）。

## 5. Phase 3：文档一致性（已完成）

本次修订目标：

1. README 与当前实现对齐。
2. 增加 Codex CLI 配置与验证步骤。
3. 固化握手链路：`initialize -> notifications/initialized -> tools/list -> tools/call`。
4. 明确 batch 不支持与 initialized notification 排障。
5. 清理旧链路文案（`zotero-mcp-server`、`23119`、`node + index.js`）。

## 6. Phase 4：本地回归与发布准备（待执行）

仅本地执行，不发布、不推送。

本地回归清单：

1. `npm run build`（`zotero-mcp-plugin`）。
2. 安装最新本地 `.xpi` 并重启 Zotero。
3. 验证 `/mcp` 基础握手链路。
4. 通过 Codex CLI 执行一次 `tools/list` 与一次工具调用。
5. 通过 Claude Code 做一次回归验证（统一 headers 配置）。

完成上述清单后，再决定是否进入发布流程。
