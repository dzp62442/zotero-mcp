# Codex 支持开发进度

更新时间：2026-03-05

## 0. 最新检查点（会话压缩后）

1. 已生成并交付本地构建插件包：`.scaffold/build/zotero-mcp-plugin.xpi`。
2. 用户已反馈：已安装新插件并重启 Zotero。
3. Phase 2 第一版代码已完成（工作区未提交）：
   - `clientConfigGenerator.ts` 新增 `codex` 客户端与 TOML 输出能力
   - 设置页客户端下拉新增 Codex
   - 新增中英文 Codex 指引与文案 key
4. Phase 1 运行态验收已完成（2026-03-05）：
   - `GET /mcp` = 200
   - `initialize` = 200
   - `notifications/initialized` = 202 + 空 body
   - `tools/list` = 200

## 1. 里程碑总览

| Phase | 目标 | 状态 | 说明 |
|---|---|---|---|
| Phase 1 | 协议兼容修复（握手） | 已完成 | 真实端到端握手与关键错误路径已验证 |
| Phase 2 | Codex 配置生成与 UI | 进行中 | 代码已落地，待构建验证与提交 |
| Phase 3 | 文档一致性修复 | 未开始 | 待更新 README/README-zh 的接入与排障章节 |
| Phase 4 | 发布与回归 | 未开始 | 待完成回归矩阵与发布清单 |

## 2. 已完成工作

### 2.1 文档与评审

1. 完成 Codex 支持方案与 issue 调研文档。
2. 完成专家评审，并将评审结论落到执行文档（单一 notification 契约、batch 策略、发布清单）。

相关提交：

1. `50477bc` `docs: add codex support review and execution plan`

### 2.2 Phase 1 协议代码实现（第一版）

1. `streamableMCPServer.ts` 已支持：
   - `notifications/initialized`
   - `initialized` 兼容路径
2. `MCPRequest.id` 改为可选（支持 notification）。
3. notification 路径返回 `202 Accepted` + 空 body。
4. 显式拒绝 batch JSON-RPC 请求，返回 `-32600`。
5. `supportedMethods` 已补充 `notifications/initialized`。
6. `mcpTest.ts` 已新增 notification / 缺失 id / batch 用例。

相关提交：

1. `50e2811` `fix: support codex initialized notification handshake`

### 2.3 Phase 2 配置与 UI 实现（工作区）

1. `clientConfigGenerator.ts` 新增 `codex` 客户端模板。
2. 新增 TOML 输出分支（`renderConfig`），支持为 Codex 生成 `[mcp_servers."..."]` 配置片段。
3. `generateFullGuide` 代码块语言支持动态切换（`json`/`toml`）。
4. `preferences.xhtml` 客户端下拉新增 Codex 选项。
5. `addon/locale/en-US` 与 `addon/locale/zh-CN` 补充 Codex 文案与配置指引。

### 2.4 Phase 1 运行态测试记录（2026-03-05）

1. `GET /mcp` -> `200`
2. `POST initialize` -> `200`（返回 `result`）
3. `POST notifications/initialized` -> `202`（`Content-Length: 0`）
4. `POST tools/list` -> `200`（返回工具列表）
5. `POST initialized`（legacy）-> `200`（`success=true`）
6. `POST tools/list`（无 `id`）-> JSON-RPC `-32600`（HTTP `200`）
7. `POST` batch 数组 -> HTTP `400` + JSON-RPC `-32600`
8. `GET /mcp/status` -> `supportedMethods` 包含 `notifications/initialized`

### 2.5 Phase 1 一致性修复（工作区）

1. `streamableMCPServer.ts` 已增加 HTTP 状态映射：
   - JSON-RPC `-32600` / `-32700` 映射为 HTTP `400`
2. 目的：统一“无 `id` request”场景在运行态与测试中的传输语义。
3. 当前状态：代码已完成并可构建，待安装新构建包后做运行态复验。

## 3. 当前阻塞与风险

### 3.1 协议语义一致性风险

1. 已在工作区修复“无 `id` request 返回 HTTP `200`”问题。
2. 风险收敛条件：安装新构建包后，运行态复验确认该场景返回 HTTP `400`。

### 3.2 本地环境依赖风险

1. 代理与沙箱权限会影响 `npm install` / `npm run build`。
2. `npm test` 需要显式提供 `ZOTERO_PLUGIN_ZOTERO_BIN_PATH`。
3. 运行时可能受 `DISPLAY`/图形会话影响。
4. Codex CLI 上游配置格式可能变更，需要在发布前做一次真实版本复核。
5. 沙箱内连接 `127.0.0.1` 偶发失败，握手验收建议在非沙箱环境执行。

## 4. 下一步执行计划（按优先级）

1. 对 Phase 2 改动执行构建与手工验证：
   - 生成 `codex` 配置时输出 TOML
   - 指引命令与 issue #24 建议命令一致
2. 安装新构建包并复验“无 `id` request”场景为 HTTP `400`。
3. 提交 Phase 2 + 协议一致性修复代码（独立 commit）。
4. Phase 3 文档同步：
   - README/README-zh 增加 Codex 接入与排障
   - 清理旧链路（`node + index.js`、旧端口描述）

## 5. 验收清单（当前快照）

| 验收项 | 当前状态 |
|---|---|
| `notifications/initialized` 代码支持 | 已完成 |
| notification `202 + 空体` 契约 | 已完成 |
| batch `-32600` 策略 | 已完成 |
| 单元级测试补充 | 已完成 |
| 真实端到端握手 smoke | 已完成 |
| Codex 配置生成/UI | 进行中（代码已完成，待验证/提交） |
