# Fork Delta Notes (vs Upstream)

更新时间：2026-03-05

## 1. 范围说明

本文件是当前 fork 的唯一汇总文档，聚焦：

1. 相对上游的改动汇总
2. 设计逻辑与关键决策
3. 本地调试指令与执行过程
4. Claude Code 启动 `failed` bug 的根因与修复

不记录任何个人 Zotero 数据库测试内容。

## 2. 改动总览（相对上游）

### 2.1 功能与协议改动

1. 补齐 `notifications/initialized` 握手路径。
2. 保留 `initialized` 兼容路径。
3. notification 语义固定为：HTTP `202` + 空 body。
4. 结构错误统一：`-32600/-32700` 映射 HTTP `400`。
5. batch JSON-RPC 数组请求显式不支持，返回 `-32600`。

### 2.2 Codex 接入改动

1. 配置生成器新增 `codex` 客户端。
2. 生成 TOML 配置（`[mcp_servers."..."]`）。
3. 默认包含 headers：`Content-Type: application/json`。
4. 设置页客户端顺序：`Claude Code` 第一，`Codex CLI` 第二。
5. 中英文 i18n 补齐，支持与 Claude Code / cc-switch 统一配置。

### 2.3 Claude 启动失败修复

1. 修复 `/mcp` 请求体截取逻辑：按 `Content-Length` 截断当前请求 body。
2. `/mcp` 不再宣称 keep-alive（与当前单请求连接实现一致）。
3. `status` 能力输出同步：`keepAliveSupported=false`。

## 3. 关键提交

1. `50e2811` `fix: support codex initialized notification handshake`
2. `b8d879f` `feat: add codex client config and align invalid request status`
3. `09a55fd` `fix: align codex order and header compatibility with claude config`
4. `2ade97a` `docs: complete phase3 codex onboarding and status sync`
5. `50fd107` `fix: harden mcp request parsing for claude startup reconnect`

## 4. 设计逻辑（决策理由）

1. 协议优先：先保证握手和错误语义一致，再做配置/UI。
2. 向后兼容：保留 `initialized`，新增 `notifications/initialized`。
3. 语义明确：notification 不返回 JSON-RPC body，避免客户端误判。
4. 配置统一：Codex 与 Claude Code 共用 HTTP endpoint + header 策略。
5. 最小改动：不做架构重写，仅修协议层、配置层、文档层。

## 5. 本地调试指令

### 5.1 依赖与构建

```bash
cd /home/dzp62442/Libraries/zotero-mcp/zotero-mcp-plugin
npm install
npm run build
```

构建产物：

- `.scaffold/build/zotero-mcp-plugin.xpi`

### 5.2 开发运行与测试

```bash
# 启动开发模式（需要 Zotero 可执行路径）
ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm run start

# 测试
CI=1 ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm test -- --no-watch
```

### 5.3 MCP 握手检查

```bash
curl -i http://127.0.0.1:23120/mcp
curl -sS http://127.0.0.1:23120/mcp/status
```

关键期望：

1. `initialize` -> HTTP `200`
2. `notifications/initialized` -> HTTP `202` + 空 body
3. `tools/list` -> HTTP `200`
4. 缺失 `id` 的 request -> HTTP `400` + `-32600`

## 6. 调试过程（摘要）

1. 先确认 Codex 链路：`initialize -> notifications/initialized -> tools/list`。
2. 完成 Codex 配置生成、UI 顺序、headers 兼容。
3. 文档对齐到插件集成 HTTP 架构（清理旧链路/旧端口文案）。
4. 发现 Claude Code 启动首连 `failed`、手动 reconnect 后恢复。
5. 复现实验确认请求边界问题与连接语义不一致问题。
6. 应用修复后重建本地包并复测，Claude 启动恢复正常。

## 7. Claude Code 启动失败 bug 复盘

### 7.1 现象

1. Codex 启动正常。
2. Claude Code 启动时 `zotero-mcp` 显示 `failed`。
3. 手动 `reconnect` 后可正常使用。

### 7.2 根因

1. `/mcp` POST body 提取未按 `Content-Length` 截断。
2. 当同连接上后续请求字节被拼入当前 body 时，会触发 `-32700 Parse error`。
3. 服务端曾返回 `Connection: keep-alive`，但实现实际会在 finally 关闭流，语义不一致。

### 7.3 修复

涉及文件：

1. `zotero-mcp-plugin/src/modules/httpServer.ts`
2. `zotero-mcp-plugin/src/modules/streamableMCPServer.ts`

修复点：

1. 仅消费当前请求 body（按 `Content-Length`）。
2. `/mcp` 不再宣称 keep-alive。
3. `keepAliveSupported` 改为 `false`。

### 7.4 验证

1. 本地 `npm run build` 通过。
2. 握手与错误路径检查通过。
3. 用户侧验证：Claude Code 启动“似乎没有问题了”。

## 8. 当前状态与后续建议

当前状态：

1. 本地代码与文档已收敛。
2. 未发布、未推送。

建议（可选）：

1. 增加一条 HTTP 集成回归测试，覆盖“同连接连续请求”场景。
2. 在发布前做一次 Codex/Claude 双客户端回归。
