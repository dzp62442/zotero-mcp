# Codex 支持调试手册（Runbook）

更新时间：2026-03-05

## 1. 目的

这份手册用于统一以下任务的排查路径：

1. 本地依赖安装与构建
2. 测试启动与 Zotero 可执行路径配置
3. MCP 握手 smoke 测试（`initialize -> notifications/initialized`）
4. 常见失败（代理、沙箱、端口未监听、测试为 0 passed）

## 2. 环境前置检查

在 `zotero-mcp-plugin` 目录执行：

```bash
node -v
npm -v
npm config get registry
npm config list -l | rg -n "proxy|https-proxy|registry"
```

建议期望：

1. Node 版本满足脚手架要求（`zotero-plugin-scaffold` 需要较新 Node）。
2. 代理配置与实际网络环境一致。

## 3. 依赖与构建

### 3.1 安装依赖

```bash
cd /home/dzp62442/Libraries/zotero-mcp/zotero-mcp-plugin
npm install
```

成功信号：

1. 出现 `added XXX packages`
2. 无 `connect EPERM 127.0.0.1:7890` 类错误

### 3.2 构建

```bash
npm run build
```

成功信号：

1. `Build finished`
2. `tsc --noEmit` 无报错退出

## 4. 测试执行

### 4.1 设置 Zotero 可执行路径

Linux 示例：

```bash
export ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/home/dzp62442/Softwares/Zotero_linux-x86_64/zotero
```

运行：

```bash
npm test -- --no-watch
```

可选 headless：

```bash
CI=1 npm test -- --no-watch
```

注意：

1. 当前仓库未建立 `test/` 用例目录时，可能出现 `0 passed`，这不代表握手逻辑已验证。
2. 测试前建议关闭手动启动的 Zotero，避免实例冲突。

## 5. 握手 Smoke 测试（强制执行）

确保 Zotero 中插件已启用服务器，端口为 `23120`。

### 5.1 检查端口

```bash
curl -i http://127.0.0.1:23120/mcp
```

期望：

1. 返回 `200` 且有 endpoint 信息。

### 5.2 initialize

```bash
curl -i -X POST http://127.0.0.1:23120/mcp \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"manual-test","version":"0.1"}}}'
```

期望：

1. `200`
2. 返回 `result.protocolVersion` / `capabilities` / `serverInfo`

### 5.3 notifications/initialized

```bash
curl -i -X POST http://127.0.0.1:23120/mcp \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'
```

期望：

1. `202 Accepted`
2. 空 body（`Content-Length: 0`）

### 5.4 tools/list

```bash
curl -i -X POST http://127.0.0.1:23120/mcp \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

期望：

1. `200`
2. 返回工具列表

## 6. 常见故障与定位

### 6.1 `connect EPERM 127.0.0.1:7890`

含义：

1. npm 代理连接失败（环境/权限/沙箱）。

处理：

1. 确认本机代理服务与端口状态。
2. 若在受限沙箱执行，需切换到可访问网络环境。

### 6.2 `No Zotero Found`

含义：

1. `ZOTERO_PLUGIN_ZOTERO_BIN_PATH` 未设置或路径无效。

处理：

1. 显式设置环境变量并确认文件可执行。

### 6.3 `Unable to connect to Zotero` / `ECONNREFUSED`

含义：

1. 测试 runner 未连接到目标 Zotero 调试端口。

处理：

1. 先关闭手动启动的 Zotero 再跑测试。
2. 使用 `CI=1` 尝试 headless。
3. 提高日志级别：`ZOTERO_PLUGIN_LOG_LEVEL=trace`。

### 6.4 `curl 127.0.0.1:23120/mcp` 连接拒绝

含义：

1. 插件 MCP 服务器未监听目标端口。

处理：

1. 在 Zotero UI 中确认插件已启用服务。
2. 检查配置项：
   - `extensions.zotero.zotero-mcp-plugin.mcp.server.enabled=true`
   - `extensions.zotero.zotero-mcp-plugin.mcp.server.port=23120`
3. 复查插件启动日志，确认 `HttpServer.start()` 被调用。

## 7. 推荐执行顺序（最短路径）

1. `npm install`
2. `npm run build`
3. 设置 `ZOTERO_PLUGIN_ZOTERO_BIN_PATH` 后执行 `npm test -- --no-watch`
4. 手动完成第 5 节四条 curl 握手测试
5. 将命令输出归档到 issue 或 PR
