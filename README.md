# Zotero MCP

Zotero MCP integrates Zotero with MCP clients through a built-in Streamable HTTP server inside the Zotero plugin.

_This README is also available in: [:cn: 简体中文](./README-zh.md) | :gb: English._

## Overview

- Architecture: `MCP Client <-> HTTP /mcp <-> Zotero MCP Plugin <-> Zotero Library`
- No separate `zotero-mcp-server` process is required.
- Default endpoint: `http://127.0.0.1:23120/mcp`

## Quick Start

1. Download the latest `zotero-mcp-plugin-*.xpi` from [Releases](https://github.com/cookjohn/zotero-mcp/releases).
2. Install it in Zotero: `Tools -> Add-ons -> Install Add-on From File`.
3. Restart Zotero.
4. Open `Preferences -> Zotero MCP Plugin`:
- Enable Server
- Keep/update port (default `23120`)
- Use "Generate Client Configuration"

## Codex CLI Setup

### Method 1: CLI (recommended)

```bash
codex mcp add zotero-mcp http://127.0.0.1:23120/mcp -t http
```

### Method 2: `~/.codex/config.toml`

```toml
[mcp_servers."zotero-mcp"]
type = "http"
url = "http://127.0.0.1:23120/mcp"

[mcp_servers."zotero-mcp".headers]
"Content-Type" = "application/json"
```

### Compatibility with Claude Code / cc-switch

Codex supports the same endpoint + header strategy used by Claude Code HTTP config:

- same URL: `http://127.0.0.1:23120/mcp`
- same header: `Content-Type: application/json`

## Claude Code (HTTP)

```bash
claude mcp add --transport http zotero-mcp http://127.0.0.1:23120/mcp
```

JSON example:

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

## Protocol Notes

Expected handshake chain:

1. `initialize`
2. `notifications/initialized`
3. `tools/list`
4. `tools/call`

Current transport behavior:

- `notifications/initialized` (no `id`) -> HTTP `202`, empty body (`Content-Length: 0`)
- invalid single request (`-32600`) -> HTTP `400`
- parse error (`-32700`) -> HTTP `400`
- batch JSON-RPC array requests are not supported -> `-32600`

## Verification

Basic endpoint checks:

```bash
curl -i http://127.0.0.1:23120/mcp
curl -sS http://127.0.0.1:23120/mcp/status
```

Codex checks:

```bash
codex mcp list
codex mcp get zotero-mcp
```

## Troubleshooting

### `Transport channel closed, when send initialized notification`

- Confirm plugin server is enabled in Zotero preferences.
- Confirm you are using endpoint `/mcp` (not old stdio config).
- Confirm request chain includes `initialize -> notifications/initialized`.
- Confirm plugin version includes Codex compatibility fixes (v1.4.0+).

### `Connection refused` / timeout

- Ensure Zotero is running.
- Ensure port matches plugin settings (default `23120`).
- Check local proxy/VPN rules for `127.0.0.1`.

### Invalid request errors

- Notification methods must be sent without `id`.
- Non-notification methods must include valid `id`.
- Batch request arrays are intentionally unsupported.

## Developer Setup

Requirements:

- Zotero 7+
- Node.js 18+
- npm

Build:

```bash
cd zotero-mcp-plugin
npm install
npm run build
```

The built package is generated at:

- `.scaffold/build/zotero-mcp-plugin.xpi`

Dev run:

```bash
ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm run start
```

Tests:

```bash
CI=1 ZOTERO_PLUGIN_ZOTERO_BIN_PATH=/path/to/zotero npm test -- --no-watch
```

## Key MCP Tools

- `search_library`
- `get_item_details`
- `get_content`
- `get_annotations`
- `get_collections`
- `search_fulltext`
- `semantic_search`
- `semantic_status`

## License

[MIT](./LICENSE)
