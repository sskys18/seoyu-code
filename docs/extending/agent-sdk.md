# Agent SDK

The Agent SDK lets you drive Claude Code programmatically — running queries, managing sessions, registering in-process tools, and wiring up hook callbacks — without launching an interactive terminal session.

The public API is exported from `src/entrypoints/agentSdkTypes.ts`, which re-exports from three sub-modules:

| Module | Contents |
|--------|----------|
| `sdk/coreTypes.ts` | Serializable message and config types |
| `sdk/runtimeTypes.ts` | Non-serializable types (callbacks, session handles) |
| `sdk/controlTypes.ts` | Control-protocol types for SDK builders |

## Core API

### `query(params)`

Single-turn or multi-turn query entry point. Returns an async iterable of `SDKMessage` objects:

```typescript
import { query } from '@anthropic-ai/claude-code'

const stream = query({
  prompt: 'List files in the current directory',
  options: {
    cwd: '/my/project',
    model: 'claude-opus-4-6',
    maxTurns: 5,
  },
})

for await (const message of stream) {
  if (message.type === 'assistant') {
    console.log(message.message.content)
  }
  if (message.type === 'result') {
    console.log('Cost:', message.total_cost_usd)
  }
}
```

`Options` fields (selected):

| Field | Type | Description |
|-------|------|-------------|
| `cwd` | string | Working directory for the session |
| `model` | string | Model ID (e.g. `claude-opus-4-6`) |
| `maxTurns` | number | Maximum agentic turns before stopping |
| `systemPrompt` | string | Override the system prompt |
| `appendSystemPrompt` | string | Append to the default system prompt |
| `mcpServers` | record | Additional MCP server configs for this query |
| `tools` | `SdkMcpToolDefinition[]` | In-process tool implementations |
| `permissionMode` | string | `default`, `acceptEdits`, `bypassPermissions`, `plan` |
| `includeHookEvents` | boolean | Emit hook execution events in the stream |
| `thinking` | `ThinkingConfig` | Extended thinking configuration |
| `outputFormat` | `OutputFormat` | Structured output schema |

### `unstable_v2_createSession(options)` (alpha)

Creates a persistent session for multi-turn conversations. The session maintains conversation history across multiple `turn()` calls:

```typescript
import { unstable_v2_createSession } from '@anthropic-ai/claude-code'

const session = unstable_v2_createSession({
  cwd: '/my/project',
  model: 'claude-opus-4-6',
})

// First turn
for await (const msg of session.turn('Hello')) {
  // handle messages
}

// Second turn — continues the same conversation
for await (const msg of session.turn('Now write tests for that')) {
  // handle messages
}

await session.close()
```

### `unstable_v2_prompt(message, options)` (alpha)

One-shot convenience function for single prompts. Awaits the full result:

```typescript
const result = await unstable_v2_prompt('What files are here?', {
  cwd: '/my/project',
  model: 'claude-sonnet-4-6',
})
console.log(result.result)
```

## Session Management

### `listSessions(options?)`

Returns metadata for all sessions, optionally filtered by project directory:

```typescript
const sessions = await listSessions({ dir: '/my/project' })
for (const s of sessions) {
  console.log(s.sessionId, s.summary, s.createdAt)
}
```

`SDKSessionInfo` fields: `sessionId`, `summary`, `title`, `tag`, `projectPath`, `createdAt`, `updatedAt`, `isActive`.

### `getSessionInfo(sessionId, options?)`

Reads metadata for a single session without scanning all sessions.

### `getSessionMessages(sessionId, options?)`

Returns the full conversation transcript as `SessionMessage[]`, optionally filtered by `limit`, `offset`, and `includeSystemMessages`.

### `renameSession(sessionId, title, options?)`

Appends a custom-title entry to the session's JSONL transcript.

### `tagSession(sessionId, tag, options?)`

Tags a session. Pass `null` to clear the tag.

### `forkSession(sessionId, options?)`

Creates a new session branching from an existing one. Supports `upToMessageId` for branching from a specific point. Returns `{ sessionId }` for the new session.

## In-Process Tools

The `tool()` helper and `createSdkMcpServer()` let you expose custom tools that run in the same Node.js process as the SDK consumer, without spawning a separate MCP server process:

```typescript
import { tool, createSdkMcpServer } from '@anthropic-ai/claude-code'
import { z } from 'zod'

const myTool = tool(
  'get_weather',
  'Fetches current weather for a city',
  {
    city: z.string().describe('City name'),
  },
  async ({ city }) => {
    const data = await fetchWeather(city)
    return { content: [{ type: 'text', text: JSON.stringify(data) }] }
  },
)

const server = createSdkMcpServer({
  name: 'my-tools',
  version: '1.0.0',
  tools: [myTool],
})

// Pass to query options
const stream = query({
  prompt: 'What is the weather in Paris?',
  options: { mcpServers: { 'my-tools': server } },
})
```

The `SdkMcpToolDefinition` carries a Zod schema for input validation and a handler function. Tool annotations (`readOnly`, `destructive`, `openWorld`) can be set via the `extras` parameter.

## Message Types (SDKMessage)

All streamed messages conform to the `SDKMessage` discriminated union:

| `type` | Description |
|--------|-------------|
| `user` | User turn content |
| `assistant` | Assistant turn content (may include tool uses) |
| `tool_result` | Tool execution result |
| `system` | System messages (slash commands, session events) |
| `result` | Final result with cost, usage, and stop reason |

`SDKResultMessage` (the `result` type) includes:

```typescript
{
  type: 'result'
  subtype: 'success' | 'error_max_turns' | 'error_during_execution'
  duration_ms: number
  duration_api_ms: number
  is_error: boolean
  num_turns: number
  result: string              // Final text output
  total_cost_usd: number
  usage: ModelUsage
  session_id: string
  stop_reason: string | null
  permission_denials: PermissionDenial[]
}
```

## Hook Callbacks via SDK

The SDK control protocol supports registering hook callbacks that fire as named JavaScript functions instead of shell commands. Register them in the `initialize` control request:

```typescript
// From the control schema:
{
  subtype: 'initialize',
  hooks: {
    PreToolUse: [{
      matcher: 'Bash',
      hookCallbackIds: ['my-bash-guard'],
      timeout: 30
    }]
  }
}
```

When the `PreToolUse` event fires for a `Bash` call, the runtime invokes the registered callback identified by `my-bash-guard` and waits for its response before proceeding.

## Thinking Configuration

Control the model's extended thinking:

```typescript
// Adaptive (Opus 4.6+) — model decides when and how much to think
options: { thinking: { type: 'adaptive' } }

// Fixed budget
options: { thinking: { type: 'enabled', budgetTokens: 10000 } }

// Disabled
options: { thinking: { type: 'disabled' } }
```

## Output Format

Request structured JSON output matching a JSON Schema:

```typescript
options: {
  outputFormat: {
    type: 'json_schema',
    schema: {
      type: 'object',
      properties: {
        files: { type: 'array', items: { type: 'string' } }
      }
    }
  }
}
```

The model's final response will be valid JSON conforming to the schema.

## MCP Server Configuration

Pass additional MCP servers scoped to a single query:

```typescript
options: {
  mcpServers: {
    'my-server': {
      type: 'stdio',
      command: 'npx',
      args: ['-y', 'my-mcp-package'],
      env: { API_KEY: process.env.API_KEY }
    },
    'remote-server': {
      type: 'http',
      url: 'https://my-mcp-server.example.com/mcp',
      headers: { Authorization: `Bearer ${token}` }
    }
  }
}
```

Supported transport types: `stdio`, `sse`, `http`, `ws`, `sdk` (in-process).

## Control Protocol (for SDK Builders)

SDK implementations (e.g., the Python SDK) communicate with the CLI process over a stdio control channel. The protocol consists of `SDKControlRequest` and `SDKControlResponse` messages.

Request subtypes:

| Subtype | Description |
|---------|-------------|
| `initialize` | Start session, register hooks and MCP servers |
| `interrupt` | Abort the current turn |
| `set_model` | Change the model mid-session |
| `set_max_thinking_tokens` | Adjust thinking budget |
| `set_permission_mode` | Change tool permission mode |
| `mcp_status` | Query MCP server connection states |
| `get_context_usage` | Get token usage breakdown by category |
| `can_use_tool` | Permission check for a tool call |

The `initialize` response returns available commands, models, account info, and the CLI process PID (used by the IDE bridge for tmux socket isolation).

## Remote Control (Internal)

For daemon architectures that keep a persistent bridge to claude.ai, `connectRemoteControl` establishes a WebSocket connection owned by the parent process rather than the agent subprocess. This allows the daemon to survive agent crashes and respawn while claude.ai maintains the same session.

This API is marked `@internal` and requires pre-authorization; see `src/assistant/daemonBridge.ts` for the full integration pattern.

## Beta Features

The `betas` option enables preview API features:

```typescript
options: {
  betas: ['context-1m-2025-08-07']  // 1M token context window
}
```
