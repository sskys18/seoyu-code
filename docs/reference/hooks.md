# Hooks Reference

All React hooks in `src/hooks/`. Most hooks are consumed by the main REPL component (`src/screens/REPL.tsx`) or by other hooks.

| Hook Name | File | Purpose | Key Dependencies |
|-----------|------|---------|-----------------|
| `useApiKeyVerification` | `useApiKeyVerification.ts` | Verifies Anthropic API key validity; provides reverify callback | `services/api/claude`, `utils/auth` |
| `useArrowKeyHistory` | `useArrowKeyHistory.tsx` | Up/down arrow navigation through prompt history with chunked disk loading | `history.ts`, `ink.useInput`, `PromptInput/inputModes` |
| `useAssistantHistory` | `useAssistantHistory.ts` | Fetches and paginates remote assistant session history | `assistant/sessionHistory`, `remote/sdkMessageAdapter` |
| `useAwaySummary` | `useAwaySummary.ts` | Generates an away summary after 5 min of terminal blur (AWAY_SUMMARY flag) | `services/awaySummary`, `ink/terminal-focus-state` |
| `useBackgroundTaskNavigation` | `useBackgroundTaskNavigation.ts` | Keyboard navigation between foreground/background agent tasks | `state/AppState`, `tasks/InProcessTeammateTask`, `ink.useInput` |
| `useBlink` | `useBlink.ts` | Synchronized blinking animation that pauses when offscreen | `ink.useAnimationFrame`, `ink.useTerminalFocus` |
| `useCancelRequest` | `useCancelRequest.ts` | Handles Escape / cancel keybinding to abort active requests | `state/AppState`, `useCommandQueue`, `context/overlayContext` |
| `useCanUseTool` | `useCanUseTool.tsx` | Core permission gate: checks allow/deny rules and presents approval UI | `tools/BashTool/bashPermissions`, `utils/permissions/permissions`, `Tool.ts` |
| `useChromeExtensionNotification` | `useChromeExtensionNotification.tsx` | Shows notification prompting user to install Chrome extension | Analytics, `useIssueFlagBanner` |
| `useClaudeCodeHintRecommendation` | `useClaudeCodeHintRecommendation.tsx` | Surfaces contextual hints about Claude Code features | Analytics, settings |
| `useClipboardImageHint` | `useClipboardImageHint.ts` | Detects clipboard images and shows paste hint | Platform utilities |
| `useCommandKeybindings` | `useCommandKeybindings.tsx` | Registers `command:*` keybinding actions that invoke slash commands | `keybindings/KeybindingContext`, `keybindings/useKeybinding` |
| `useCommandQueue` | `useCommandQueue.ts` | Exposes the queued-command store snapshot | `utils/messageQueueManager` |
| `useCopyOnSelect` | `useCopyOnSelect.ts` | Auto-copies selected terminal text to clipboard | Terminal utilities |
| `useDeferredHookMessages` | `useDeferredHookMessages.ts` | Manages deferred messages injected by hook callbacks | Hook messaging utilities |
| `useDiffData` | `useDiffData.ts` | Fetches git diff data (hunks + stats) for the diff viewer | `utils/gitDiff` |
| `useDiffInIDE` | `useDiffInIDE.ts` | Streams file edits to the IDE for side-by-side diff display | `services/mcp/types`, `tools/FileEditTool`, MCP IDE server |
| `useDirectConnect` | `useDirectConnect.ts` | Manages direct WebSocket connection to CCR (DIRECT_CONNECT flag) | `server/directConnectManager`, `remote/*` |
| `useDoublePress` | `useDoublePress.ts` | Detects double-press of a key within a configurable time window | Timing utilities |
| `useDynamicConfig` | `useDynamicConfig.ts` | Reads dynamic (runtime) config values from AppState | `state/AppState` |
| `useElapsedTime` | `useElapsedTime.ts` | Tracks elapsed wall-clock time with configurable update interval | React timer |
| `useExitOnCtrlCD` | `useExitOnCtrlCD.ts` | Handles Ctrl+C / Ctrl+D to exit the REPL (with double-press guard) | `ink.useInput` |
| `useExitOnCtrlCDWithKeybindings` | `useExitOnCtrlCDWithKeybindings.ts` | Keybinding-aware variant of exit handler | `keybindings/useKeybinding` |
| `useFileHistorySnapshotInit` | `useFileHistorySnapshotInit.ts` | Initializes file history snapshot on session start | File history utilities |
| `useGlobalKeybindings` | `useGlobalKeybindings.tsx` | Registers global keybindings (screenshot, scroll, panel toggle, etc.) | `keybindings/useKeybinding`, `state/AppState` |
| `useHistorySearch` | `useHistorySearch.ts` | Ctrl+R incremental history search with Emacs kill-ring support | `history.ts`, `ink.useInput`, `keybindings/useKeybinding` |
| `useIdeAtMentioned` | `useIdeAtMentioned.ts` | Tracks when IDE files are @-mentioned in the prompt | IDE utilities |
| `useIdeConnectionStatus` | `useIdeConnectionStatus.ts` | Polls MCP server connection state to detect IDE online/offline | MCP state |
| `useIDEIntegration` | `useIDEIntegration.tsx` | Initializes MCP-based IDE integration and extension install flow | `utils/ide`, `services/mcp/types` |
| `useIdeLogging` | `useIdeLogging.ts` | Forwards diagnostic events to the IDE via MCP | `services/lsp` |
| `useIdeSelection` | `useIdeSelection.ts` | Syncs IDE text selection into the prompt as context | MCP IDE server |
| `useInboxPoller` | `useInboxPoller.ts` | Polls agent message inbox for swarm/teammate communication | `state/AppState`, `tasks/InProcessTeammateTask`, `services/notifier` |
| `useInputBuffer` | `useInputBuffer.ts` | Debounced ring-buffer for prompt input state (used by undo/redo) | React state |
| `useIssueFlagBanner` | `useIssueFlagBanner.ts` | Shows banners for known issue flags (e.g. rate limit warnings) | Analytics, settings |
| `useLogMessages` | `useLogMessages.ts` | Collects and exposes internal log messages for the debug view | Log utilities |
| `useLspPluginRecommendation` | `useLspPluginRecommendation.tsx` | Surfaces LSP plugin install recommendations in the UI | `services/lsp`, plugin utilities |
| `useMailboxBridge` | `useMailboxBridge.ts` | Bridge between UDS mailbox and React state | Mailbox utilities |
| `useMainLoopModel` | `useMainLoopModel.ts` | Returns the current main-loop model identifier from state | `bootstrap/state` |
| `useManagePlugins` | `useManagePlugins.ts` | Loads/unloads plugins and reinitializes LSP on plugin changes | `utils/plugins/*`, `services/lsp/manager` |
| `useMemoryUsage` | `useMemoryUsage.ts` | Polls Node.js heap usage and reports high/critical thresholds | `usehooks-ts/useInterval` |
| `useMergedClients` | `useMergedClients.ts` | Merges multiple MCP client instances for unified access | MCP state |
| `useMergedCommands` | `useMergedCommands.ts` | Combines built-in commands with plugin-provided commands | `commands.ts`, plugin state |
| `useMergedTools` | `useMergedTools.ts` | Assembles the full tool pool (built-in + MCP + extra) with deny-rule filtering | `tools.ts/assembleToolPool`, `state/AppState` |
| `useMinDisplayTime` | `useMinDisplayTime.ts` | Enforces a minimum display duration for ephemeral UI elements | React timer |
| `useNotifyAfterTimeout` | `useNotifyAfterTimeout.ts` | Sends a system notification after a long idle/operation period | `services/notifier`, `bootstrap/state` |
| `useOfficialMarketplaceNotification` | `useOfficialMarketplaceNotification.tsx` | Shows notification about the official plugin marketplace | Analytics |
| `usePasteHandler` | `usePasteHandler.ts` | Handles text and image paste events into the prompt | `utils/imagePaste`, `utils/platform` |
| `usePluginRecommendationBase` | `usePluginRecommendationBase.tsx` | Base hook for plugin recommendation banners | Plugin utilities, analytics |
| `usePromptsFromClaudeInChrome` | `usePromptsFromClaudeInChrome.tsx` | Receives prompt injections from Claude in Chrome extension | MCP/bridge |
| `usePromptSuggestion` | `usePromptSuggestion.ts` | Fetches next-turn prompt suggestions from the suggestion service | `services/PromptSuggestion` |
| `usePrStatus` | `usePrStatus.ts` | Polls GitHub PR status (review state, number, URL) every 60 s | `utils/ghPrStatus` |
| `useQueueProcessor` | `useQueueProcessor.ts` | Drains the command queue when conditions are met (no active UI, etc.) | `utils/messageQueueManager`, `utils/queueProcessor` |
| `useRemoteSession` | `useRemoteSession.ts` | Manages CCR remote session lifecycle (connect, permission bridge, disconnect) | `remote/RemoteSessionManager`, `remote/sdkMessageAdapter` |
| `useReplBridge` | `useReplBridge.tsx` | Main bridge hook: wires REPL to remote/bridge/CCR control plane | `bridge/*`, `commands.ts`, `state/AppState` |
| `useScheduledTasks` | `useScheduledTasks.ts` | Fires scheduled cron tasks at the correct time and injects messages | `utils/cronScheduler`, `utils/cronTasks`, `Task.ts` |
| `useSearchInput` | `useSearchInput.ts` | Emacs-style search input with kill-ring support for the history search UI | `utils/Cursor`, `ink.useInput` |
| `useSessionBackgrounding` | `useSessionBackgrounding.ts` | Handles Ctrl+B to background/foreground the current query | `state/AppState` |
| `useSettings` | `useSettings.ts` | Returns current settings from AppState (reactive to file changes) | `state/AppState` |
| `useSettingsChange` | `useSettingsChange.ts` | Watches settings files for changes and updates AppState | File watch utilities |
| `useSkillImprovementSurvey` | `useSkillImprovementSurvey.ts` | Shows skill improvement survey after skill use (SKILL_IMPROVEMENT flag) | Analytics |
| `useSkillsChange` | `useSkillsChange.ts` | Watches skill directories for changes and invalidates skill caches | `skills/loadSkillsDir` |
| `useSSHSession` | `useSSHSession.ts` | REPL integration for `claude ssh` sessions via a child SSH process | `ssh/SSHSessionManager`, `remote/*` |
| `useSwarmInitialization` | `useSwarmInitialization.ts` | Initializes swarm teammate hooks and context on agent spawn/resume | `utils/swarm/*`, `utils/agentSwarmsEnabled` |
| `useSwarmPermissionPoller` | `useSwarmPermissionPoller.ts` | Polls for permission responses from team leader in worker agents | `utils/permissions/PermissionUpdateSchema`, `usehooks-ts/useInterval` |
| `useTaskListWatcher` | `useTaskListWatcher.ts` | Watches the task list directory for external changes (debounced) | `utils/tasks`, `fs.watch` |
| `useTasksV2` | `useTasksV2.ts` | Reactive task list for TodoV2 with fs.watch + fallback polling | `utils/tasks`, `state/AppState` |
| `useTeammateViewAutoExit` | `useTeammateViewAutoExit.ts` | Automatically exits teammate view when the active teammate finishes | `state/AppState`, teammate utilities |
| `useTeleportResume` | `useTeleportResume.tsx` | Resumes a teleported remote code session | `utils/teleport`, `utils/conversationRecovery` |
| `useTerminalSize` | `useTerminalSize.ts` | Returns current terminal column/row dimensions | `ink` terminal utilities |
| `useTextInput` | `useTextInput.ts` | Core text input state machine (cursor, kill-ring, history insertion) | `utils/Cursor`, `history.ts`, `ink.Key` |
| `useTimeout` | `useTimeout.ts` | One-shot timeout that clears on unmount | React timer |
| `useTurnDiffs` | `useTurnDiffs.ts` | Computes per-turn file diffs for the diff viewer | `utils/gitDiff` |
| `useTypeahead` | `useTypeahead.tsx` | Full typeahead/autocomplete for slash commands, @-mentions, and files | `commands.ts`, `state/AppState`, `keybindings/*` |
| `useUpdateNotification` | `useUpdateNotification.ts` | Shows a banner when a new Claude Code version is available | Version check utilities |
| `useVimInput` | `useVimInput.ts` | Vim modal editing state machine for the prompt input | `vim/operators`, `vim/transitions`, `utils/Cursor` |
| `useVirtualScroll` | `useVirtualScroll.ts` | Virtualized scrolling for large transcript lists | `ink/components/ScrollBox`, `useDeferredValue` |
| `useVoice` | `useVoice.ts` | Hold-to-talk voice input via Anthropic voice_stream STT | `services/voiceStreamSTT`, `services/voiceKeyterms`, native audio |
| `useVoiceEnabled` | `useVoiceEnabled.ts` | Returns whether voice input is enabled for the current session | `voice/voiceModeEnabled`, settings |
| `useVoiceIntegration` | `useVoiceIntegration.tsx` | Integrates voice recording into the REPL input flow (VOICE_MODE flag) | `useVoice`, `keybindings/*`, `context/voice` |
