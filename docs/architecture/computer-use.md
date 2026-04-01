# Computer Use

## 1. Purpose

The computer-use subsystem gives Claude Code the ability to control the macOS GUI: take screenshots, move the mouse, click, type, and manage application windows. It is implemented as an MCP server (answering `ListTools` and `CallTool` from the model) backed by two native modules: `@ant/computer-use-input` (Rust/enigo for mouse and keyboard) and `@ant/computer-use-swift` (Swift for SCContentFilter screenshots and NSWorkspace app enumeration). The feature is gated by the `tengu_malort_pedway` GrowthBook flag and the `CHICAGO_MCP` bundle feature flag.

## 2. Key Files

| File | Size | Role |
|------|------|------|
| `src/utils/computerUse/wrapper.tsx` | 48.3 KB | `.call()` override; binds `ComputerUseSessionContext` to each tool invocation |
| `src/utils/computerUse/executor.ts` | 23.3 KB | `ComputerExecutor` impl: mouse, keyboard, clipboard, screenshots |
| `src/utils/computerUse/toolRendering.tsx` | 17.3 KB | React rendering overrides for computer-use tool results in the REPL UI |
| `src/utils/computerUse/mcpServer.ts` | 4.0 KB | In-process MCP server factory; standalone stdio entrypoint |
| `src/utils/computerUse/hostAdapter.ts` | 2.7 KB | Process-lifetime `ComputerUseHostAdapter` singleton |
| `src/utils/computerUse/computerUseLock.ts` | 7.0 KB | Cross-session mutex preventing concurrent computer-use |
| `src/utils/computerUse/appNames.ts` | 6.4 KB | App enumeration and filtering for tool descriptions |
| `src/utils/computerUse/gates.ts` | 2.5 KB | Feature gate helpers (`getChicagoEnabled`, `getChicagoSubGates`) |
| `src/utils/computerUse/cleanup.ts` | 3.2 KB | Post-turn cleanup: unhide windows, release state |
| `src/utils/computerUse/escHotkey.ts` | 1.9 KB | ESC hotkey registration to abort in-flight computer-use |
| `src/utils/computerUse/drainRunLoop.ts` | 2.8 KB | macOS run-loop pump for async input dispatch |
| `src/utils/computerUse/common.ts` | 2.6 KB | Shared constants (`CLI_HOST_BUNDLE_ID`, capabilities) |
| `src/utils/computerUse/swiftLoader.ts` | 925 B | Lazy loader for `@ant/computer-use-swift` native module |
| `src/utils/computerUse/inputLoader.ts` | 1.2 KB | Lazy loader for `@ant/computer-use-input` native module |
| `src/utils/computerUse/setup.ts` | 2.0 KB | One-time permission and hardware setup |

## 3. Data Flow

```mermaid
flowchart TD
    Model -->|tool_use: computer_use_*| CU[MCP CallTool handler in client.ts]
    CU -->|feature CHICAGO_MCP| W[wrapper.tsx .call override]
    W --> Lock{computerUseLock}
    Lock -->|not held| Bind[bindSessionContext → dispatch]
    Lock -->|held by another session| Err[error: in use]
    Bind --> Exec[executor.ts — ComputerExecutor]
    Exec -->|mouse/keyboard| Input[@ant/computer-use-input Rust/enigo]
    Exec -->|screenshot| Swift[@ant/computer-use-swift SCContentFilter]
    Exec -->|clipboard| PB[pbcopy / pbpaste]
    Exec --> Result[CuCallToolResult]
    Result --> TR[toolRendering.tsx — React JSX for REPL display]
    TR --> REPL[REPL UI]

    MCP[MCP ListTools] -->|lazy, on first CU connection| Server[mcpServer.ts createComputerUseMcpServerForCli]
    Server --> HostAdapter[hostAdapter.ts singleton]
    HostAdapter --> Exec
    Server -->|installed apps ≤1s timeout| AppNames[appNames.ts filterAppsForDescription]
    AppNames --> ToolDesc[tool description includes app list]
```

### Lock lifecycle

The `computerUseLock` is a cross-session mutex (keyed by `sessionId`). `tryAcquireComputerUseLock()` is called at the start of each tool dispatch in `wrapper.tsx`. If another session holds it, the call returns an error immediately without blocking. The lock is released by `cleanup.ts` at the end of each turn (which also unhides any windows that were hidden before a screenshot).

## 4. Core Types

### Host adapter

```typescript
// @ant/computer-use-mcp/types (external package)
export type ComputerUseHostAdapter = {
  serverName: string
  logger: Logger
  executor: ComputerExecutor
  ensureOsPermissions(): Promise<
    { granted: true } |
    { granted: false; accessibility: boolean; screenRecording: boolean }
  >
  isDisabled(): boolean
  getSubGates(): ComputerUseSubGates
  getAutoUnhideEnabled(): boolean
  cropRawPatch(): null  // CLI: pixel-validation not available (no Electron)
}
```

### Session context (wrapper.tsx → bindSessionContext)

```typescript
// Assembled in wrapper.tsx buildSessionContext()
type ComputerUseSessionContext = {
  getAllowedApps(): string[]
  getGrantFlags(): GrantFlags
  getUserDeniedBundleIds(): string[]
  getSelectedDisplayId(): number | undefined
  getDisplayPinnedByModel(): boolean
  getLastScreenshotDims(): ScreenshotDims | undefined
  // Write-backs into AppState:
  setAllowedApps(apps: string[]): void
  setPinnedDisplay(id: number): void
  setLastScreenshotDims(dims: ScreenshotDims): void
  // Permission / UI:
  requestPermission(req: CuPermissionRequest): Promise<CuPermissionResponse>
  setToolJSX(jsx: React.ReactNode): void
  sendOSNotification(title: string, body: string): void
  abortController: AbortController
}
```

### Executor capabilities

```typescript
// src/utils/computerUse/common.ts
export const CLI_CU_CAPABILITIES = {
  screenshot: true,
  mouse: true,
  keyboard: true,
  clipboard: true,
  applications: true,
  hostBundleId: CLI_HOST_BUNDLE_ID,  // sentinel; terminal bundle ID is separate
}
```

## 5. Integration Points

| Subsystem | How it connects |
|-----------|-----------------|
| **MCP client** | `client.ts` lazily `await import()`s the computer-use module on first connection, gated on `feature('CHICAGO_MCP')`; the `.call()` override from `wrapper.tsx` is spread into the MCP tool object |
| **AppState** | `computerUseMcpState` slice holds `allowedApps`, `grantFlags`, `selectedDisplayId`, `lastScreenshotDims`; `wrapper.tsx` reads and writes through per-call context getters/setters |
| **Permission UI** | Computer use tool calls that require user approval render via `ComputerUseApproval` component (imported in `wrapper.tsx`); the same `setToolJSX` hook used by all tool permission UIs |
| **Tool rendering** | `toolRendering.tsx` provides `getComputerUseMCPRenderingOverrides()` — a map from tool name to React renderer — consumed by `client.ts` alongside Chrome rendering overrides |
| **Cleanup** | `cleanup.ts` is called by the REPL at turn end; it always calls `unhide` to restore any windows hidden before screenshots, regardless of user preference |
| **ESC abort** | `registerEscHotkey()` installs a CGEventTap that fires `abortController.abort()` when ESC is pressed during an in-flight computer-use operation |
| **Analytics / gates** | `getChicagoEnabled()` → GrowthBook `tengu_malort_pedway`; sub-gates control `mouseAnimation`, `hideBeforeAction`, `pixelValidation` independently |

## 6. Design Decisions

**Singleton host adapter.** `getComputerUseHostAdapter()` builds and caches the adapter once per process. Native modules (`@ant/computer-use-input` and `@ant/computer-use-swift`) are loaded here; failure throws immediately (no degraded mode) because there is no meaningful fallback when the hardware interface is unavailable.

**Terminal as surrogate host.** Unlike Cowork (Electron), the CLI runs inside a terminal emulator. `getTerminalBundleId()` detects the emulator and passes it as `hostBundleId` to the Swift layer, so the terminal is exempted from the hide-before-screenshot list and the z-order walk. The sentinel `CLI_HOST_BUNDLE_ID` is kept for the package's frontmost gate; the terminal being frontmost is acceptable.

**Clipboard via `pbcopy`/`pbpaste`.** The Electron `clipboard` module is unavailable. Long text inputs use `typeViaClipboard`: save existing clipboard → write text → read-back verify → Cmd+V → sleep 100 ms → restore. The read-back verify prevents a silent `pbcopy` failure from pasting stale clipboard content.

**1-second app enumeration timeout.** `tryGetInstalledAppNames()` races Spotlight enumeration against a 1 s timer. If enumeration is slow, the tool description omits the app list (soft fail); the model can still control apps by name — it just lacks the hint list.

**`cropRawPatch` returns null.** The pixel-validation feature (comparing pre/post-click screenshot patches) requires synchronous JPEG decode+crop. The CLI has only async `sharp`-compatible image processing; returning `null` skips validation. The sub-gate defaults to `false` so this path is not exercised in production.

**Cross-session lock.** Computer use is inherently stateful (mouse position, focused window). Allowing two sessions to issue mouse/keyboard events concurrently would produce unpredictable results. The cross-session lock enforces serial access; the error message tells the user which session holds the lock so they can `/exit` it.

**`drainRunLoop` for async input.** macOS input dispatch (`enigo`) posts events to the main queue. The CLI's Node event loop does not pump the CFRunLoop. `drainRunLoop` runs a short CFRunLoop tick after each input operation to flush the event queue before the next `await` boundary resolves.
