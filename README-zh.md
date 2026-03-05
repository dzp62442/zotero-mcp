# Zotero MCP

Zotero MCP 通过 Zotero 插件内置的 Streamable HTTP MCP 服务，让 AI 客户端直接访问本地 Zotero 文献库。

_This README is also available in: [:gb: English](./README.md) | :cn: 简体中文._

## 项目概览

- 架构：`MCP 客户端 <-> HTTP /mcp <-> Zotero MCP 插件 <-> Zotero 文献库`
- 不再需要单独运行 `zotero-mcp-server` 进程。
- 默认地址：`http://127.0.0.1:23120/mcp`

## 快速开始

1. 在 [Releases](https://github.com/cookjohn/zotero-mcp/releases) 下载最新 `zotero-mcp-plugin-*.xpi`。
2. 在 Zotero 中安装：`工具 -> 附加组件 -> 从文件安装附加组件`。
3. 重启 Zotero。
4. 打开 `首选项 -> Zotero MCP Plugin`：
- 勾选 `启用服务器`
- 确认端口（默认 `23120`）
- 使用“生成客户端配置”

## Codex CLI 配置

### 方法 1：命令行（推荐）

```bash
codex mcp add zotero-mcp http://127.0.0.1:23120/mcp -t http
```

### 方法 2：`~/.codex/config.toml`

```toml
[mcp_servers."zotero-mcp"]
type = "http"
url = "http://127.0.0.1:23120/mcp"

[mcp_servers."zotero-mcp".headers]
"Content-Type" = "application/json"
```

### 与 Claude Code / cc-switch 统一配置

Codex 可复用 Claude Code 的 HTTP 端点与 headers 策略：

- 同一 URL：`http://127.0.0.1:23120/mcp`
- 同一 header：`Content-Type: application/json`

## Claude Code（HTTP）配置

```bash
claude mcp add --transport http zotero-mcp http://127.0.0.1:23120/mcp
```

JSON 示例：

```json
{
  "mcpServers": {
    "zotero-mcp": {
      "type": "http",
      "url": "http://127.0.0.1:23120/mcp",
      "headers": {
        "Content-Type": "application/json"
      }
    }
  }
}
```

## 协议行为说明

标准握手链路：

1. `initialize`
2. `notifications/initialized`
3. `tools/list`
4. `tools/call`

当前实现语义：

- `notifications/initialized`（无 `id`）-> HTTP `202`，空 body（`Content-Length: 0`）
- 非法单请求（`-32600`）-> HTTP `400`
- 解析错误（`-32700`）-> HTTP `400`
- 不支持 batch JSON-RPC 数组请求 -> `-32600`

## 连通性验证

基础检查：

```bash
curl -i http://127.0.0.1:23120/mcp
curl -sS http://127.0.0.1:23120/mcp/status
```

Codex 检查：

```bash
codex mcp list
codex mcp get zotero-mcp
```

## 故障排查

### `Transport channel closed, when send initialized notification`

- 确认 Zotero 插件服务已启用。
- 确认客户端连的是 `/mcp`（不是旧的 stdio 配置）。
- 确认握手顺序为 `initialize -> notifications/initialized`。
- 确认插件版本包含 Codex 兼容修复（v1.4.0+）。

### `Connection refused` / 超时

- 确认 Zotero 正在运行。
- 确认端口与插件设置一致（默认 `23120`）。
- 检查本地代理/VPN 是否影响 `127.0.0.1`。

### Invalid request 错误

- Notification 方法必须不带 `id`。
- 非 notification 方法必须带有效 `id`。
- batch 数组请求当前为显式不支持。

## 开发者环境

依赖要求：

- Zotero 7+
- Node.js 18+
- npm

构建：

```bash
cd zotero-mcp-plugin
npm install
npm run build
```

构建产物：

- `.scaffold/build/zotero-mcp-plugin.xpi`

开发启动：

```bash
ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm run start
```

测试：

```bash
CI=1 ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm test -- --no-watch
```

## 主要 MCP 工具

- `search_library`
- `get_item_details`
- `get_content`
- `get_annotations`
- `get_collections`
- `search_fulltext`
- `semantic_search`
- `semantic_status`

## 许可证

[MIT](./LICENSE)
