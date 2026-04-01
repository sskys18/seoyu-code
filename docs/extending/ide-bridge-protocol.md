# IDE Bridge Protocol

The bridge connects the Claude Code CLI to IDE extensions (VS Code, JetBrains, etc.) and to the claude.ai web app. It uses a WebSocket-based bidirectional protocol layered on top of JWT-authenticated sessions.

## Architecture Overview

```
IDE Extension / claude.ai web
        |  WebSocket (WSS)
        v
  Bridge Server (claude.ai infrastructure)
        |  WebSocket (WSS) + poll loop
        v
  bridgeMain.ts  (bridge daemon process)
        |  stdio (spawn subprocess)
        v
  Claude Code REPL  (agent process)
        |
  replBridge.ts / replBridgeTransport.ts
        |  WebSocket (WSS, from REPL side)
        v
  Bridge Server
```

There are two bridging modes:

- **REPL bridge** (`replBridge.ts`) — The REPL opens its own WebSocket to the bridge server and sends messages directly. Used when `claude --enable-remote-control` is active.
- **Standalone bridge** (`bridgeMain.ts`) — A separate daemon process polls for work assignments, spawns REPL subprocesses per session, and handles reconnection. Used by `claude --bridge` (the remote-development mode for IDE extensions).

## JWT Handshake

All bridge connections are authenticated with short-lived JWTs issued by the claude.ai backend. The token lifecycle:

1. **Initial token** — Obtained via the claude.ai OAuth flow and stored in the system keychain. The CLI reads this at startup.
2. **Session ingress token** — A per-session JWT issued by the bridge server when a session is created (`POST /v1/code/sessions`). Prefixed with `sk-ant-si-`.
3. **Proactive refresh** — `createTokenRefreshScheduler` decodes the JWT's `exp` claim and schedules a refresh 5 minutes before expiry. If the token is not a decodable JWT (e.g. an OAuth token), the refresh chain is preserved from the last scheduled timer.
4. **Fallback refresh** — After a successful refresh, a follow-up timer fires after 30 minutes regardless of the new token's expiry, preventing long-running sessions from going stale.

JWT decoding (without signature verification) is performed by `decodeJwtPayload` in `src/bridge/jwtUtils.ts`:

```typescript
function decodeJwtPayload(token: string): unknown | null
```

Tokens prefixed with `sk-ant-si-` have that prefix stripped before base64url-decoding the payload segment.

The refresh scheduler supports up to 3 consecutive failures before stopping, and schedules a 60-second retry delay on each failure.

## Message Types

All messages are JSON objects. The bridge protocol distinguishes three categories:

### SDK Messages (`SDKMessage`)

The core message stream. Discriminated on the `type` field:

| `type` | Direction | Description |
|--------|-----------|-------------|
| `user` | inbound | User prompt from the IDE/web |
| `assistant` | outbound | Model response turn |
| `tool_result` | outbound | Tool execution result |
| `system` | outbound | Slash command events, session metadata |
| `result` | outbound | Final result with cost and usage |

Every message carrying a `uuid` field is deduplicated by the bridge layer using bounded ring buffers (`BoundedUUIDSet`). This prevents double-processing of replayed messages after transport reconnection.

### Control Requests (`SDKControlRequest`)

Server-to-CLI control messages. Must be responded to promptly (10-14 second server timeout):

```typescript
type SDKControlRequest = {
  type: 'control_request'
  request_id: string
  request: {
    subtype: 'initialize' | 'interrupt' | 'set_model' |
             'set_max_thinking_tokens' | 'set_permission_mode' |
             'can_use_tool' | 'mcp_status' | 'get_context_usage'
    // ... subtype-specific fields
  }
}
```

### Control Responses (`SDKControlResponse`)

CLI-to-server acknowledgements:

```typescript
type SDKControlResponse = {
  type: 'control_response'
  response: {
    subtype: 'success' | 'error'
    request_id: string
    response?: object   // subtype-specific data
    error?: string      // on subtype='error'
  }
}
```

## Message Routing in the REPL Bridge

`handleIngressMessage` in `src/bridge/bridgeMessaging.ts` is the central ingress router:

1. Parse the raw WebSocket message as JSON.
2. Check for `control_response` — route to `onPermissionResponse`.
3. Check for `control_request` — route to `handleServerControlRequest`.
4. Validate as `SDKMessage` via type predicate.
5. Check UUID against `recentPostedUUIDs` (echo of our own sent messages) and `recentInboundUUIDs` (re-delivery after reconnect).
6. Route `user` messages to `onInboundMessage`; ignore other inbound types.

`handleServerControlRequest` responds to each control subtype:

| Subtype | REPL bridge response |
|---------|---------------------|
| `initialize` | Returns empty capabilities, process PID |
| `set_model` | Calls `onSetModel` callback, returns success |
| `set_max_thinking_tokens` | Calls `onSetMaxThinkingTokens`, returns success |
| `set_permission_mode` | Calls `onSetPermissionMode`; may return error if disabled |
| `interrupt` | Calls `onInterrupt`, returns success |
| Unknown | Returns error so server doesn't hang |

Outbound-only mode (the `outboundOnly` flag) rejects all mutable requests with an error except `initialize`, which must always succeed.

## Bridge Transport

`ReplBridgeTransport` (`src/bridge/replBridgeTransport.ts`) abstracts the underlying WebSocket connection:

- Maintains reconnect logic with exponential backoff.
- Tracks `lastTransportSequenceNum` to resume from the correct position after reconnect (prevents history replay of already-delivered messages).
- Buffers outbound messages during reconnection.
- Detects system sleep/wake events by comparing elapsed wall time to expected poll intervals (threshold = 2× the connection backoff cap).

## Eligible Bridge Messages

Not all REPL internal messages are forwarded to the bridge. `isEligibleBridgeMessage` filters:

- `user` messages — forwarded (unless `isVirtual`)
- `assistant` messages — forwarded (unless `isVirtual`)
- `system` messages with `subtype === 'local_command'` — forwarded
- All others (tool results, progress, etc.) — dropped

Virtual messages (from inner REPL calls) are display-only and not exposed to bridge consumers; they see the summarizing `tool_use`/`tool_result` pair instead.

## Session Archival

When a bridge session ends, the CLI sends a minimal `result` message before closing the WebSocket. This triggers server-side session archival. The `makeResultMessage` helper in `bridgeMessaging.ts` constructs this message with zero-cost/zero-usage fields.

## Echo Deduplication (`BoundedUUIDSet`)

The ring-buffer set tracks recently seen UUIDs with O(1) add/has and O(capacity) memory:

```typescript
class BoundedUUIDSet {
  constructor(capacity: number)
  add(uuid: string): void
  has(uuid: string): boolean
  clear(): void
}
```

Two sets are maintained per connection: `recentPostedUUIDs` (messages we sent, to ignore echoes) and `recentInboundUUIDs` (messages we forwarded, to ignore replays). Default capacity is large enough to cover a full history replay window.

## Bridge Daemon (Standalone Mode)

`runBridgeLoop` in `bridgeMain.ts` is the main loop for the standalone daemon:

1. **Registration** — Registers the worker with the bridge server (`registerWorker`), receiving an `environmentId` and `environmentSecret`.
2. **Poll loop** — Polls for work assignments from the bridge server. Assignments contain a session ID, ingress token, and optional worktree configuration.
3. **Session spawn** — For each new assignment, `SessionSpawner.spawn` creates a new REPL subprocess with `--sdk-url` pointing at the bridge server. The subprocess connects its own WebSocket.
4. **Heartbeat** — Periodically heartbeats all active sessions to keep them alive on the server.
5. **Backoff** — Uses exponential backoff with jitter on connection errors. A separate backoff budget applies to general API errors vs. connection errors.
6. **Worktree management** — For sessions requiring isolated Git worktrees, `createAgentWorktree` creates a new worktree and `removeAgentWorktree` cleans it up after the session ends.

The daemon supports multiple concurrent sessions (`--spawn N` or `--capacity`) gated behind the `tengu_ccr_bridge_multi_session` feature flag.

## Building a New IDE Extension

To connect a new IDE to Claude Code:

### 1. Use the MCP SSE-IDE Transport

The simplest integration: launch `claude` with MCP configuration pointing at a local SSE server provided by your extension:

```json
{
  "mcpServers": {
    "my-ide": {
      "type": "sse-ide",
      "url": "http://localhost:{port}/sse",
      "ideName": "my-ide",
      "ideRunningInWindows": false
    }
  }
}
```

The `sse-ide` transport carries IDE metadata and enables path conversion (Windows ↔ POSIX) when `ideRunningInWindows` is true.

### 2. Implement the MCP Server in Your Extension

Your extension runs an MCP SSE server on a local port. Register tools that expose IDE capabilities:

- `openFile(path)` — Open a file in the editor
- `getDiagnostics(path)` — Return current lint/type errors
- `showDiff(before, after)` — Show a diff in the editor's diff viewer

Claude Code normalizes these as `my_ide__openFile`, `my_ide__getDiagnostics`, etc.

### 3. Connect via the Agent SDK

For tighter integration, use the Agent SDK to drive Claude Code programmatically from your extension process:

```typescript
import { query } from '@anthropic-ai/claude-code'

const stream = query({
  prompt: userInput,
  options: {
    cwd: workspacePath,
    mcpServers: {
      'my-ide': myInProcessMcpServer
    }
  }
})

for await (const msg of stream) {
  if (msg.type === 'assistant') {
    displayInPanel(msg.message.content)
  }
}
```

### 4. Remote Control (claude.ai Bridge)

For extensions that want the full claude.ai multi-session experience with persistent history:

1. Obtain an OAuth token via the claude.ai OAuth flow.
2. Create a session via `POST /v1/code/sessions` (claude.ai API).
3. Open a WebSocket to the session URL.
4. Forward user messages as `SDKMessage` with `type: 'user'`.
5. Receive `assistant`, `result`, and `control_request` messages from the CLI.
6. Respond to `control_request` messages (especially `initialize`) promptly.

The `SDKControlInitializeResponse` from `controlSchemas.ts` must be returned within the server's timeout window to keep the connection alive.

### Token Refresh

If your extension holds the OAuth token, implement proactive refresh using the same 5-minute-before-expiry strategy used by the CLI:

```typescript
const { schedule } = createTokenRefreshScheduler({
  getAccessToken: () => myExtension.getOAuthToken(),
  onRefresh: (sessionId, newToken) => {
    ws.send(JSON.stringify({ type: 'refresh_token', token: newToken, session_id: sessionId }))
  },
  label: 'my-ide-bridge'
})

// Call after receiving a session token
schedule(sessionId, ingressToken)
```
