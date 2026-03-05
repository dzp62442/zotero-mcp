# Codex 支持任务交接（压缩对话后继续）

更新时间：2026-03-05

## 1. 结论快照

1. Phase 1 已完成：协议握手兼容与错误语义已落地。
2. Phase 2 已完成：Codex 配置生成、UI 顺序、headers 兼容已落地。
3. Phase 3 已完成：README 与 README-zh 已同步到当前实现。
4. 当前仅剩 Phase 4 本地回归与发布前检查。

## 2. 关键实现事实

1. 握手链路：`initialize -> notifications/initialized -> tools/list -> tools/call`。
2. `notifications/initialized` 为 notification 路径，返回 `202 + 空 body`。
3. 非法请求：`-32600/-32700` 返回 HTTP `400`。
4. batch JSON-RPC 数组请求：显式不支持，返回 `-32600`。
5. Codex 配置：默认包含 `Content-Type: application/json` headers。
6. 配置 UI 顺序：`Claude Code` 第一，`Codex CLI` 第二。

## 3. 已完成提交

1. `50477bc` `docs: add codex support review and execution plan`
2. `50e2811` `fix: support codex initialized notification handshake`
3. `908c016` `docs: update codex progress and phase1 validation status`
4. `b8d879f` `feat: add codex client config and align invalid request status`
5. `09a55fd` `fix: align codex order and header compatibility with claude config`

## 4. 下一步（本地优先，不发布不推送）

1. `cd zotero-mcp-plugin && npm run build`
2. 安装 `.scaffold/build/zotero-mcp-plugin.xpi` 并重启 Zotero
3. 端点检查：`/mcp` 与 `/mcp/status`
4. Codex CLI 检查：`codex mcp list`、`codex mcp get zotero-mcp`
5. 进行一次真实 `tools/list` + 工具调用回归
6. Claude Code 做一次回归，确认统一 headers 配置可用

## 5. 排障优先顺序

1. 检查插件服务是否启用、端口是否一致（默认 `23120`）。
2. 检查客户端是否连接 `/mcp`（非旧 stdio 链路）。
3. 检查 `supportedMethods` 是否包含 `notifications/initialized`。
4. 检查是否误发 batch 数组请求。
