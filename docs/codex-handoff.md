# Codex 支持任务交接（用于压缩对话后继续）

更新时间：2026-03-05

## 1. 当前结论

1. Phase 1 代码改造已完成并提交：
   - `notifications/initialized` 支持
   - notification 语义（`202 + 空体`）
   - batch 请求返回 `-32600`
   - 对应测试补充
2. 用户已确认：已安装新插件并重启 Zotero。
3. Phase 2 第一版代码已在工作区完成（尚未提交）：
   - 新增 Codex 客户端配置生成（含 TOML 输出）
   - Codex TOML 默认包含 `headers.Content-Type`
   - 设置页顺序为：`Claude Code` 第一、`Codex CLI` 第二
   - 中英文指引文案已补齐（含 cc-switch 统一配置说明）
4. Phase 1 运行态测试已完成（2026-03-05）：
   - 握手链路通过：`GET /mcp`、`initialize`、`notifications/initialized`、`tools/list`
   - `notifications/initialized` 返回 `202 + 空 body`
   - batch 请求返回 `400 + -32600`
5. 已在工作区修复“无 `id` request 返回 HTTP 200”问题：
   - 现改为 JSON-RPC `-32600` 对应 HTTP `400`
   - 待安装新构建包后做运行态复验

## 2. 已完成提交

1. `50477bc` `docs: add codex support review and execution plan`
2. `50e2811` `fix: support codex initialized notification handshake`

## 3. 待执行（下一步优先）

1. 在 `zotero-mcp-plugin` 目录执行构建检查并提交 Phase 2 代码。
2. 安装新构建包并复验“无 `id` request”返回 HTTP `400`。
3. 提交 Phase 2 + 协议一致性修复代码。
4. 进入 Phase 3：README/README-zh 文档一致性修订。

## 4. Phase 1 测试结论（已完成）

1. `GET /mcp` => `200`
2. `initialize` => `200` + `result`
3. `notifications/initialized` => `202` + `Content-Length: 0`
4. `tools/list` => `200`
5. `initialized`（legacy）=> `200` + `success=true`
6. batch 请求 => `400 + -32600`
7. `/mcp/status` 包含 `notifications/initialized`

## 5. 若后续回归失败的排查顺序

1. `curl /mcp/status` 检查 `supportedMethods` 是否包含 `notifications/initialized`。
2. 确认加载的是新安装插件版本（避免旧实例缓存）。
3. 检查插件设置中的 server enabled 与端口配置。
4. 收集日志并对照 `docs/codex-debug-playbook.md` 第 6 节故障分支。
