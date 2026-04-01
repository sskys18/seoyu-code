# MCP Integration

Claude Code uses the Model Context Protocol (MCP) as its primary mechanism for exposing external tools to the model. All Claude Code built-in tools (Bash, Edit, Read, etc.) and user-defined tools are normalized to the same MCP tool interface internally.

## Architecture Overview

```
settings.json / SDK options
        |
        v
  MCP Config Layer (src/services/mcp/config.ts)
        |
        v
  MCPConnectionManager  ←── AppState.mcpServers
        |
        v
  Per-server client (src/services/mcp/client.ts)
        |
  Transport  ──────────────────────────────────
  stdio | SSE | HTTP | WebSocket | SDK (in-process)
```

The config layer aggregates server definitions from all settings scopes (`user`, `project`, `local`, `managed`, `dynamic`, `enterprise`, `claudeai`) into a single `ScopedMcpServerConfig` map. The connection manager diffs the desired vs. connected state and creates/tears down clients as needed.

## Server Discovery

Claude Code discovers MCP servers from multiple sources, in merge order:

1. **Policy settings** (`managed` scope) — admin-deployed, highest priority
2. **User settings** (`~/.claude/settings.json`)
3. **Project settings** (`.claude/settings.json`)
4. **Local settings** (`.claude/settings.local.json`)
5. **Plugin-provided** — from each enabled plugin's `mcp/mcp.json`
6. **SDK options** — `mcpServers` passed to `query()` or session options (`dynamic` scope)
7. **claude.ai proxy** — servers from the claude.ai web app (`claudeai` scope)

Later sources do not override earlier ones for the same server name within the same scope.

## Server Configuration

Declare MCP servers under the `mcpServers` key in any settings file:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"],
      "env": { "API_KEY": "${MY_API_KEY}" }
    }
  }
}
```

Environment variables in `env` values are expanded at connection time.

## Transport Types

### stdio

Spawns a subprocess and communicates over stdin/stdout using newline-delimited JSON-RPC. This is the most common transport for local MCP servers.

```json
{
  "type": "stdio",
  "command": "node",
  "args": ["./my-server.js"],
  "env": { "PORT": "8080" }
}
```

The subprocess inherits a sanitized environment (no sensitive shell vars) and the current working directory.

### SSE (Server-Sent Events)

Connects to a remote HTTP server using the SSE MCP transport. Supports OAuth authentication.

```json
{
  "type": "sse",
  "url": "https://my-mcp-server.example.com/sse",
  "headers": { "Authorization": "Bearer my-token" },
  "oauth": {
    "clientId": "my-client-id",
    "authServerMetadataUrl": "https://auth.example.com/.well-known/openid-configuration"
  }
}
```

### HTTP (Streamable HTTP)

Uses the MCP Streamable HTTP transport (POST-based, supports streaming responses).

```json
{
  "type": "http",
  "url": "https://my-mcp-server.example.com/mcp",
  "headers": { "X-API-Key": "secret" }
}
```

### WebSocket

Connects to a WebSocket MCP server. Supports TLS options from the mTLS configuration.

```json
{
  "type": "ws",
  "url": "wss://my-mcp-server.example.com/ws"
}
```

### SDK (In-Process)

Registers an in-process MCP server created with `createSdkMcpServer()`. The server runs in the same Node.js process as the SDK consumer. Used by the Agent SDK to expose custom tools without spawning a subprocess.

```json
{
  "type": "sdk",
  "name": "my-in-process-server"
}
```

### IDE-Specific Transports (Internal)

`sse-ide` and `ws-ide` are internal transports used by the IDE bridge to connect to language extension MCP servers. They carry additional metadata (`ideName`, `ideRunningInWindows`) used for display purposes and path conversion. These are not for external use.

## Connection Lifecycle

1. **Pending** — Server config loaded, connection not yet attempted.
2. **Connecting** — Client instantiated, transport handshake in progress.
3. **Connected** — MCP `initialize` completed, capabilities exchanged, tools available.
4. **Needs-auth** — Server returned an auth challenge; user must complete OAuth flow.
5. **Failed** — Connection error; may retry with backoff.
6. **Disabled** — Server explicitly disabled by policy.

Connection state is stored in `AppState.mcpServers` as a `MCPServerConnection` discriminated union.

## OAuth Authentication

SSE and HTTP servers can require OAuth 2.0 authentication. The flow:

1. Claude Code attempts to connect; server returns 401.
2. `handleOAuth401Error` is called, which initiates the OAuth PKCE flow.
3. A local callback server listens on `callbackPort` (default: random available port).
4. The user is directed to the authorization URL in their browser.
5. After authorization, the callback server receives the code, exchanges it for tokens, and stores them in the system keychain via `secureStorage`.
6. The connection is re-attempted with the access token in the `Authorization` header.

The `headersHelper` field in server config can specify a shell command that returns headers as JSON — useful for injecting dynamically generated tokens without storing them in config:

```json
{
  "type": "http",
  "url": "https://my-server.example.com/mcp",
  "headersHelper": "my-token-helper --format json"
}
```

### Cross-App Access (XAA)

For servers that require IdP-based SSO (enterprise environments), set `oauth.xaa: true`:

```json
{
  "type": "http",
  "url": "https://enterprise-server.example.com/mcp",
  "oauth": { "xaa": true }
}
```

XAA uses a shared IdP configuration from `settings.xaaIdp` (issuer, clientId) applied across all XAA-enabled servers.

## Tool Normalization

MCP tool names must match `^[a-zA-Z0-9_-]{1,64}$`. Claude Code normalizes server names before constructing tool names:

1. Non-alphanumeric characters (including `.` and spaces) are replaced with `_`.
2. For claude.ai server names (prefixed with `claude.ai `), consecutive underscores are collapsed and leading/trailing underscores stripped.

The full tool name exposed to the model is `{normalizedServerName}__{toolName}`, using `__` as the delimiter. For example, server `my.server` with tool `get_file` becomes `my_server__get_file`.

Original (pre-normalization) tool names are preserved in `SerializedTool.originalToolName` for round-trip accuracy.

## Exposing Tools via MCP

To expose your own tools to Claude Code, implement an MCP server:

### Minimal stdio Server (Node.js)

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'

const server = new Server({ name: 'my-tools', version: '1.0.0' }, {
  capabilities: { tools: {} }
})

server.setRequestHandler('tools/list', async () => ({
  tools: [{
    name: 'get_time',
    description: 'Returns the current UTC time',
    inputSchema: { type: 'object', properties: {} }
  }]
}))

server.setRequestHandler('tools/call', async ({ params }) => {
  if (params.name === 'get_time') {
    return { content: [{ type: 'text', text: new Date().toISOString() }] }
  }
  throw new Error(`Unknown tool: ${params.name}`)
})

await server.connect(new StdioServerTransport())
```

Register it in your settings:

```json
{
  "mcpServers": {
    "my-tools": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/my-tools-server.js"]
    }
  }
}
```

### Tool Annotations

Claude Code reads MCP tool annotations to drive permissions UI:

```typescript
{
  name: 'delete_file',
  description: 'Deletes a file',
  annotations: {
    destructive: true,    // Shows destructive warning in permissions
    readOnly: false,      // Not a read-only tool
    openWorld: false      // Does not access external resources
  }
}
```

## MCP Resources

Claude Code supports MCP resources (read-only data exposed by servers) via:
- `ListMcpResourcesTool` — lists available resources from a connected server
- `ReadMcpResourceTool` — reads a specific resource by URI

Resources appear in the model's tool list when the server declares `resources` capability.

## Elicitation

Servers can request additional information from the user via the MCP `elicitation` protocol. When a server sends an `elicitation/create` request, Claude Code shows a modal dialog. The user's response is relayed back to the server via `elicitationHandler.ts`.

## Channel Allowlist

The `claude/channel` experimental capability gates access to privileged server features. A server is only granted channel access if its providing plugin appears on the administrator-managed channel allowlist (`channelAllowlist.ts`). Presence of `capabilities.experimental['claude/channel']` in the `McpServerStatus` response indicates the server has been granted channel access.

## Large Output Handling

MCP tool results that exceed the token budget are automatically truncated or persisted to disk. Binary content (images, audio) is saved to a temp file and the model receives a reference path. The truncation logic is in `mcpValidation.ts` and output storage in `mcpOutputStorage.ts`.

## Debugging

Enable MCP debug logging with `ANTHROPIC_LOG=debug` or via the `--debug` flag. MCP-specific log messages are prefixed with `[MCP]` and written to the debug log file.

The `/mcp` command in the CLI shows connection status for all configured servers, including error messages for failed connections.
