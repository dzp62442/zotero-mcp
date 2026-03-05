# Zotero MCP 支持 Codex 开发方案（可执行版）

更新时间：2026-03-05

## 1. 目标与范围

目标：在不破坏现有 Claude/Cursor/Gemini/Qwen 等客户端兼容性的前提下，使 `zotero-mcp` 稳定支持 Codex 通过 Streamable HTTP 完成握手并可调用工具。

范围边界：

1. 本轮“必达”目标是 **Codex CLI** 稳定可用。
2. “Codex Desktop”仅在确认存在可复现接入方式后纳入；否则在文档中标注“待验证”。
3. 不做大规模架构重构，仅在协议层、配置层、文档层做最小必要改动。

## 2. 根因与现状

基于上游 issue 与当前代码现状，可确认两类阻塞：

1. 协议兼容阻塞（启动失败）
   - 服务端处理 `initialized`，但未处理 `notifications/initialized`。
   - 请求/响应类型仍以“有 id 的 request-response”为默认模型，notification 语义未被明确定义。
2. 产品接入阻塞（可用性差）
   - 配置生成器与偏好设置 UI 无 Codex 选项。
   - 用户只能手工配置，且很难定位握手问题。

## 3. 设计原则

1. 协议优先：先修复握手语义，再做配置/UI/文档。
2. 向后兼容：保留 `initialized`（历史兼容），新增 `notifications/initialized`（规范路径）。
3. 明确语义：notification 不走常规 JSON-RPC 响应路径，避免“看似成功但传输层异常”。
4. 可回归：每阶段必须有可复现脚本、最小测试矩阵和可观测日志。

## 4. Phase 1：协议兼容修复（阻塞项）

目标：Codex 握手稳定通过，不再出现 `initialized notification` 导致的连接中断。

### 4.1 协议语义（必须明确）

服务端按请求类型分流：

1. 普通请求（有 `id`）
   - 返回标准 JSON-RPC response（`result` 或 `error`）。
2. notification（无 `id`）
   - 服务端执行副作用逻辑并记录日志。
   - HTTP **固定返回** `202 Accepted` + 空 body（`Content-Length: 0`）。
   - 不返回 JSON-RPC `error`（notification 不期望响应）。
3. 非法请求
   - 对“应为 request 却缺失 id”的场景返回 `-32600`（`id: null`）。
   - 解析失败返回 `-32700`。

### 4.2 代码改动（最小且完整）

1. `zotero-mcp-plugin/src/modules/streamableMCPServer.ts`
   - `MCPRequest.id` 改为可选：`id?: string | number | null`。
   - 增加请求分类逻辑：`isNotification = !("id" in request) || request.id === null`。
   - `processRequest()` 同时支持：
     - `notifications/initialized`
     - `initialized`（兼容）
   - `initialized`/`notifications/initialized` 对 notification 路径不构造 JSON-RPC response。
   - `getStatus().supportedMethods` 增加 `notifications/initialized`。
2. `zotero-mcp-plugin/src/modules/httpServer.ts`
   - 接住 notification 无响应体语义，输出 `202 Accepted` + `Content-Length: 0`。
   - 保持现有 session 与 keep-alive 行为，不额外引入破坏性变更。
3. 类型统一
   - 响应类型需允许“无 JSON-RPC body”的 HTTP 返回模型，避免仅靠 `MCPResponse` 强行表示 notification。

### 4.3 测试与验收

新增最小测试集（必须包含）：

1. 单元级（`mcpTest.ts`）
   - `initialize` 正常响应。
   - `notifications/initialized`（无 `id`）可处理且不报错。
   - `initialized`（有 `id`）保持兼容。
   - 非法方法在 request/notification 两种路径语义正确。
2. 传输级（HTTP）
   - 通过 `/mcp` 发起 `initialize -> notifications/initialized -> tools/list -> tools/call` 全链路。
   - 验证 notification 返回符合预期：状态码 `202`、空 body、`Content-Length: 0`。

### 4.4 Batch 请求策略（显式决策）

当前版本明确策略如下：

1. `/mcp` 仅支持单个 JSON-RPC 对象请求。
2. 若收到 JSON 数组（batch 请求），返回 `-32600 Invalid Request`（`id: null`）。
3. 在 README 故障排查中明确“当前不支持 batch”，避免客户端误用。

验收门槛：

1. Codex CLI 接入后稳定握手，不再出现：
   - `Transport channel closed, when send initialized notification`
2. `tools/list`、`tools/call` 可持续成功。
3. Claude Code +（Gemini CLI 或 Qwen Code）回归通过。
4. 非法 batch 请求行为与文档一致（`-32600`）。

## 5. Phase 2：Codex 配置生成与 UI 支持

目标：用户无需手写配置即可完成 Codex 接入。

### 5.1 代码改动

1. `zotero-mcp-plugin/src/modules/clientConfigGenerator.ts`
   - 新增 `codex` 客户端类型。
   - 补充 Codex CLI 指引与命令示例。
   - 若需输出 TOML，需补充“非 JSON 输出模式”能力（当前 `generateConfig` 默认 `JSON.stringify`，需显式设计兼容方案）。
2. `zotero-mcp-plugin/addon/content/preferences.xhtml`
   - 客户端下拉新增 Codex。
3. i18n
   - `addon/locale/en-US/preferences.ftl`
   - `addon/locale/zh-CN/preferences.ftl`
   - 增加 Codex 相关文本 key。

### 5.2 验收标准

1. UI 可选 Codex，点击生成后输出结构正确。
2. 指引命令在当前版本 Codex CLI 上实测可用（在文档标注验证日期）。
3. 中英文文本无缺失 key。

## 6. Phase 3：文档一致性修复

目标：让 README 与真实行为一致，降低重复报错。

更新文件：

1. `README.md`
2. `README-zh.md`

内容要求：

1. 新增 Codex 接入章节（CLI + 配置文件方式）。
2. 明确握手链路：`initialize -> notifications/initialized -> tools/list/tools/call`。
3. 故障排查新增 `initialized notification` 专项条目。
4. 必须清理与“集成 HTTP 架构”不一致的旧排查内容：
   - 旧 `node + index.js` 链路
   - 旧 `zotero-mcp-server` 目录链路
   - 旧端口 `23119` 相关描述（统一到当前端口策略）

验收标准：

1. 用户按文档步骤可直接完成接入。
2. 文档命令、配置字段、端口示例与当前实现一致。

## 7. Phase 4：发布与回归

目标：可安全发布并关闭相关 issue。

流程建议：

1. 在 fork 分支完成 Phase 1~3。
2. 回归矩阵（最小）：
   - Codex CLI（必测）
   - Claude Code（回归）
   - Gemini CLI 或 Qwen Code（回归）
3. PR 描述中明确：
   - 根因
   - 协议语义决策（notification 响应）
   - 回归结果
4. 关联上游 issue：#24 #35 #36。

发布检查清单（必须满足）：

1. 版本与分发元数据同步更新：
   - `zotero-mcp-plugin/package.json`
   - `zotero-mcp-plugin/update.json`
   - `zotero-mcp-plugin/update-beta.json`（如发布 beta）
2. 变更日志明确包含：
   - `notifications/initialized` 兼容修复
   - Codex 配置生成支持
   - 文档迁移与旧链路清理
3. 发布门槛：
   - Codex CLI 通过为硬门槛
   - Codex Desktop 未验证不阻塞本次发布，但需在 release note 标注“待验证”

## 8. 风险与缓解

1. 风险：notification 返回语义实现不一致导致客户端兼容抖动。
   - 缓解：在方案中固定语义并写入测试。
2. 风险：Codex 配置格式/命令变更。
   - 缓解：文档标注“验证日期 + 版本”；生成器集中维护。
3. 风险：README 与 UI 引导不一致。
   - 缓解：同一 PR 内同步改代码与文档。

## 9. 交付拆分建议

1. Commit A：Phase 1 协议修复 + 测试
2. Commit B：Phase 2 Codex 配置生成 + UI + i18n
3. Commit C：Phase 3 文档更新

每个 commit 保持“可单独 review、可单独验证”。
