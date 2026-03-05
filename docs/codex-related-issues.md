# Codex Related Issues (Upstream: cookjohn/zotero-mcp)

Last updated: 2026-03-05 (verified via GitHub MCP)

## Issue List

## 1) #24 `codex cli`
- Link: https://github.com/cookjohn/zotero-mcp/issues/24
- State: Open
- Core ask:
  - Add Codex CLI configuration support in client configuration generator / docs.
- Meaning for implementation:
  - This is the product-level request (UI + generated config + user guidance).

## 2) #35 `Codex cli error`
- Link: https://github.com/cookjohn/zotero-mcp/issues/35
- State: Open
- Error symptom:
  - `Transport channel closed, when send initialized notification`
- Reporter config:
  - Codex server type `http`, URL `http://127.0.0.1:23120/mcp`
- Meaning for implementation:
  - This is runtime handshake failure, not only config template absence.

Additional evidence from issue comments:
- Comment link: https://github.com/cookjohn/zotero-mcp/issues/35#issuecomment-3978948223
- Repro chain in comment:
  - `GET /mcp` works
  - `initialize` works
  - `notifications/initialized` returns `-32601`
  - `tools/list` works in same session
- Implementation meaning:
  - Failure is concentrated in lifecycle notification handling, not endpoint availability or tools implementation.

## 3) #36 `MCP notifications/initialized 握手失败`
- Link: https://github.com/cookjohn/zotero-mcp/issues/36
- State: Open
- Core diagnosis in issue:
  - `initialize` succeeds.
  - `notifications/initialized` currently returns method-not-found.
  - `tools/list` and other methods can work.
- Suggested fix in issue:
  - Support both `notifications/initialized` and `initialized`.
  - Allow request `id` to be optional for notifications.
  - Handle notification response semantics correctly.
- Meaning for implementation:
  - This is the protocol compatibility root-cause issue for Codex startup.

## Code Mapping (Current Main)

- `zotero-mcp-plugin/src/modules/streamableMCPServer.ts`
  - `MCPRequest.id` is mandatory (`id: string | number`).
  - `MCPResponse.id` is mandatory (`id: string | number`), which conflicts with notification no-response semantics.
  - `processRequest()` handles `initialized` but not `notifications/initialized`.
  - Unknown methods return `-32601`.
  - `getStatus().supportedMethods` includes `initialized`, not `notifications/initialized`.
- `zotero-mcp-plugin/src/modules/clientConfigGenerator.ts`
  - No Codex client config template or guide.
- `zotero-mcp-plugin/addon/content/preferences.xhtml`
  - No Codex option in the client type dropdown.

## Issue -> Task Mapping

- #36 + #35 -> Phase 1 (protocol and handshake compatibility).
- #24 -> Phase 2 (Codex config generator + UI + setup instructions).
- All three -> Phase 3/4 (docs consistency + regression tests + release verification).

## Gaps Found During Review

1. Notification HTTP behavior must be fixed to one contract (single status/body semantics), not multiple acceptable variants.
2. Batch JSON-RPC request policy must be explicit (supported vs rejected) and reflected in tests/docs.
3. Release tasks should include version/update manifest synchronization, not only code + README updates.
