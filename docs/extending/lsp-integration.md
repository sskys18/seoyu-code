# LSP Integration

Claude Code integrates with Language Server Protocol (LSP) servers to provide real-time code intelligence. Diagnostics from language servers (type errors, lint warnings, etc.) are fed into the model's context automatically, allowing Claude to understand and fix issues without running separate lint or build commands.

## Architecture

LSP servers in Claude Code are exclusively provided by plugins — they cannot be configured directly in `settings.json`. The integration layer:

```
Plugin lsp/lsp.json
        |
        v
  getAllLspServers()           (src/services/lsp/config.ts)
        |
        v
  LSPServerManager            (src/services/lsp/LSPServerManager.ts)
  - extension → server map
  - per-file routing
        |
        v
  LSPServerInstance           (src/services/lsp/LSPServerInstance.ts)
  - per-server state
  - document sync
        |
        v
  LSPClient                   (src/services/lsp/LSPClient.ts)
  - vscode-jsonrpc transport
  - stdio subprocess
```

The manager routes all file operations to the appropriate server based on file extension.

## Providing an LSP Server via a Plugin

Create `lsp/lsp.json` inside your plugin directory:

```json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".ts": "typescript",
        ".tsx": "typescriptreact",
        ".js": "javascript",
        ".jsx": "javascriptreact"
      },
      "initializationOptions": {
        "preferences": {
          "includeInlayParameterNameHints": "all"
        }
      }
    }
  }
}
```

`ScopedLspServerConfig` fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | yes | LSP server executable |
| `args` | string[] | no | Command arguments |
| `env` | record | no | Extra environment variables |
| `extensionToLanguage` | record | yes | Map from file extension to LSP language ID |
| `initializationOptions` | object | no | Passed to the server's `initialize` request |
| `workspaceFolders` | string[] | no | Override workspace folders |

The plugin name is prepended to the server name (`{pluginName}/{serverName}`) to avoid collisions across plugins.

## LSPClient Interface

`createLSPClient` returns a low-level client wrapper around a language server subprocess:

```typescript
type LSPClient = {
  readonly capabilities: ServerCapabilities | undefined
  readonly isInitialized: boolean
  start(command: string, args: string[], options?: {
    env?: Record<string, string>
    cwd?: string
  }): Promise<void>
  initialize(params: InitializeParams): Promise<InitializeResult>
  sendRequest<TResult>(method: string, params: unknown): Promise<TResult>
  sendNotification(method: string, params: unknown): Promise<void>
  onNotification(method: string, handler: (params: unknown) => void): void
  onRequest<TParams, TResult>(
    method: string,
    handler: (params: TParams) => TResult | Promise<TResult>
  ): void
  stop(): Promise<void>
}
```

The client communicates via vscode-jsonrpc over stdio pipes. It handles pending handler registration (handlers added before the connection is ready are replayed once the connection establishes) and crash detection (non-zero exit during operation triggers the `onCrash` callback).

## LSPServerManager Interface

The manager coordinates multiple server instances and routes file operations:

```typescript
type LSPServerManager = {
  initialize(): Promise<void>
  shutdown(): Promise<void>
  getServerForFile(filePath: string): LSPServerInstance | undefined
  ensureServerStarted(filePath: string): Promise<LSPServerInstance | undefined>
  sendRequest<T>(filePath: string, method: string, params: unknown): Promise<T | undefined>
  getAllServers(): Map<string, LSPServerInstance>
  openFile(filePath: string, content: string): Promise<void>
  changeFile(filePath: string, content: string): Promise<void>
  saveFile(filePath: string): Promise<void>
  closeFile(filePath: string): Promise<void>
  isFileOpen(filePath: string): boolean
}
```

File routing is based on the extension map built during `initialize()`. A file with extension `.ts` is routed to whichever server declared `".ts"` in its `extensionToLanguage` map.

## Document Synchronization

Claude Code tracks open files and sends LSP `textDocument/didOpen`, `textDocument/didChange`, `textDocument/didSave`, and `textDocument/didClose` notifications automatically when the model edits files. The document version counter increments with each change notification. This keeps the language server's view of the file in sync with disk.

The `isFileOpen` check prevents double-open notifications when the same file is accessed via different path representations.

## Diagnostics

Language servers push diagnostics via `textDocument/publishDiagnostics` notifications. Claude Code's passive feedback system (`src/services/lsp/passiveFeedback.ts`) listens for these notifications and converts them to `DiagnosticFile` objects:

```typescript
type DiagnosticFile = {
  path: string
  diagnostics: Array<{
    message: string
    severity: 'Error' | 'Warning' | 'Info' | 'Hint'
    range: {
      start: { line: number; character: number }
      end: { line: number; character: number }
    }
    source?: string
    code?: string | number
  }>
}
```

LSP severity numbers (1=Error, 2=Warning, 3=Information, 4=Hint) are mapped to the above enum. Invalid or missing severity defaults to `'Error'`.

Diagnostics are stored in the `LSPDiagnosticRegistry` (`src/services/lsp/LSPDiagnosticRegistry.ts`) and attached to the model's context when relevant files are open. This allows the model to see type errors and lint warnings without needing to run the compiler explicitly.

## Server Capabilities

After the `initialize` handshake, `LSPClient.capabilities` reflects what the server supports. Claude Code currently leverages:

- `textDocumentSync` — Document synchronization mode (full vs. incremental)
- `publishDiagnostics` — Passive diagnostic push
- `definition` — Go-to-definition (used by code navigation tools)
- `references` — Find references
- `hover` — Hover information

Capabilities are read-only after initialization. If a server reports it does not support a capability, the corresponding request is not sent.

## Startup and Lazy Initialization

LSP servers are started lazily — the server process is not spawned until a file matching its extension is first accessed (`ensureServerStarted`). This avoids startup cost for projects that do not use the relevant language.

The manager is initialized once per session via `initialize()`, which loads all plugin-provided server configurations and builds the extension map. Servers that fail to validate their config (missing `command`, empty `extensionToLanguage`) are skipped with a warning.

## Error Handling and Crash Recovery

If a server process crashes unexpectedly (non-zero exit code while operating), the `onCrash` callback on the `LSPClient` is called. The `LSPServerInstance` marks itself as crashed, and subsequent requests to that server return `undefined` rather than throwing. The server will be restarted on the next `ensureServerStarted` call.

Plugin loading errors during `getAllLspServers()` are isolated — a failure in one plugin does not prevent other plugins from contributing their LSP servers.

## Building a Plugin with LSP Support

1. Implement a standard LSP server (any language). The server must communicate via stdio JSON-RPC.
2. Add `lsp/lsp.json` to your plugin directory.
3. Declare the file extensions your server handles.
4. The server will receive standard `initialize`, `initialized`, `textDocument/didOpen`, and `textDocument/publishDiagnostics` calls from Claude Code.

Example plugin layout:

```
my-plugin/
  manifest.json
  lsp/
    lsp.json
  bin/
    my-language-server    # Your LSP server binary
```

`lsp/lsp.json`:

```json
{
  "lspServers": {
    "my-language": {
      "command": "./bin/my-language-server",
      "args": ["--stdio"],
      "extensionToLanguage": {
        ".mylang": "my-language"
      }
    }
  }
}
```

The `command` path is resolved relative to the plugin's root directory.
