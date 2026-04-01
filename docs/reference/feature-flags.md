# Feature Flags Reference

All feature flags found by searching `feature('...')` calls across `src/`. Flags are evaluated at build time via `bun:bundle`. Some flags also gate commands, tools, or behavior at runtime.

| Flag Name | What It Enables | Gated Commands / Tools |
|-----------|-----------------|------------------------|
| `ABLATION_BASELINE` | Baseline ablation testing mode | Internal query behavior |
| `AGENT_MEMORY_SNAPSHOT` | Periodic agent memory snapshots | AgentTool memory capture |
| `AGENT_TRIGGERS` | Scheduled cron triggers for agents | `CronCreate`, `CronDelete`, `CronList` tools |
| `AGENT_TRIGGERS_REMOTE` | Remote trigger endpoints for agents | `RemoteTrigger` tool |
| `ALLOW_TEST_VERSIONS` | Permit non-release model versions | Model selection |
| `ANTI_DISTILLATION_CC` | Anti-distillation protections | System prompt encoding |
| `AUTO_THEME` | Automatic terminal theme detection | Theme service |
| `AWAY_SUMMARY` | Generate summaries when terminal is idle | `useAwaySummary` hook |
| `BASH_CLASSIFIER` | ML-based bash command classifier for permissions | BashTool permission checks |
| `BG_SESSIONS` | Background session support | Task summary module; background agent lifecycle |
| `BREAK_CACHE_COMMAND` | Enable `/break-cache` internal command | `break-cache` command (internal) |
| `BRIDGE_MODE` | Bridge mode for remote/mobile control | `bridge` command; `remote-control-server` command |
| `BUDDY` | Buddy agent system | `buddy` command |
| `BUILDING_CLAUDE_APPS` | Claude Apps building features | App scaffolding prompts |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in explore/plan agent definitions | AgentTool built-in agents |
| `BYOC_ENVIRONMENT_RUNNER` | Bring-your-own-compute environment | Remote runner selection |
| `CACHED_MICROCOMPACT` | Cache-aware micro-compaction | Compact service; context management |
| `CCR_AUTO_CONNECT` | Auto-connect to Claude Code Remote | Remote session initialization |
| `CCR_MIRROR` | Mirror session to Claude Code Remote | Session mirroring |
| `CCR_REMOTE_SETUP` | Remote environment setup UI | `remote-setup` command |
| `CHICAGO_MCP` | Enhanced MCP server integration | MCP client; analytics metadata |
| `COMMIT_ATTRIBUTION` | Attribution text in commit messages | Commit/PR commands |
| `COMPACTION_REMINDERS` | Show reminders when context is near limit | Compact warning hook |
| `CONNECTOR_TEXT` | Connector integration text | System prompts |
| `CONTEXT_COLLAPSE` | Context collapse / smart truncation | `CtxInspectTool`; query.ts context management |
| `COORDINATOR_MODE` | Coordinator+worker multi-agent mode | AgentTool coordinator routing; coordinator commands |
| `COWORKER_TYPE_TELEMETRY` | Log coworker type in analytics | Analytics metadata |
| `DAEMON` | Daemon/server process mode | `remote-control-server` command (with BRIDGE_MODE) |
| `DIRECT_CONNECT` | Direct WebSocket connection to CCR | `useDirectConnect` hook |
| `DOWNLOAD_USER_SETTINGS` | Download settings from Anthropic API | Settings sync service |
| `DUMP_SYSTEM_PROMPT` | Allow dumping the system prompt | `dumpPrompts` API call |
| `ENHANCED_TELEMETRY_BETA` | Opt-in enhanced telemetry | Analytics sink |
| `EXPERIMENTAL_SKILL_SEARCH` | Local skill search index | `ToolSearch` tool; `clearSkillIndexCache` |
| `EXTRACT_MEMORIES` | Extract durable memories at end of session | Memory extraction service |
| `FILE_PERSISTENCE` | Persist files across sessions | File state caching |
| `FORK_SUBAGENT` | Fork current session as a subagent | `fork` command; `AgentTool` fork routing |
| `HARD_FAIL` | Hard-fail on permission denials | Permission checking |
| `HISTORY_PICKER` | Enhanced history picker UI | `useArrowKeyHistory` hook |
| `HISTORY_SNIP` | Snip/trim conversation history | `SnipTool`; `force-snip` command (internal) |
| `HOOK_PROMPTS` | Inject prompts via hooks | Hook system |
| `IS_LIBC_GLIBC` | Runtime glibc detection | Native binary selection |
| `IS_LIBC_MUSL` | Runtime musl detection | Native binary selection |
| `KAIROS` | Kairos async/proactive agent system | `SleepTool`, `SendUserFileTool`, `PushNotificationTool`; `proactive`, `brief`, `assistant` commands |
| `KAIROS_BRIEF` | Kairos brief notifications only | `BriefTool`; `brief` command |
| `KAIROS_CHANNELS` | Kairos notification channels | Channel notification service |
| `KAIROS_DREAM` | Kairos dream/consolidation mode | Auto-dream service |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub PR webhook subscriptions | `SubscribePRTool`; `subscribe-pr` command (internal) |
| `KAIROS_PUSH_NOTIFICATION` | Push notifications (without full Kairos) | `PushNotificationTool` |
| `LODESTONE` | Lodestone context anchoring | Context management |
| `MCP_RICH_OUTPUT` | Rich output formatting for MCP tools | MCP client output rendering |
| `MCP_SKILLS` | MCP-provided skills as slash commands | `getMcpSkillCommands()` in commands.ts |
| `MEMORY_SHAPE_TELEMETRY` | Log memory shape/structure in telemetry | Analytics metadata |
| `MESSAGE_ACTIONS` | Message action buttons in UI | Message rendering |
| `MONITOR_TOOL` | Background monitoring tool | `MonitorTool` |
| `NATIVE_CLIENT_ATTESTATION` | Native client identity attestation | OAuth/auth headers |
| `NATIVE_CLIPBOARD_IMAGE` | Native clipboard image support | `usePasteHandler` hook |
| `NEW_INIT` | New initialization flow | Init command |
| `OVERFLOW_TEST_TOOL` | Overflow testing tool | `OverflowTestTool` |
| `PERFETTO_TRACING` | Perfetto performance tracing | Diagnostics |
| `POWERSHELL_AUTO_MODE` | Auto-detect and use PowerShell on Windows | PowerShell tool mode |
| `PROACTIVE` | Proactive agent mode | `SleepTool`; `proactive` command |
| `PROMPT_CACHE_BREAK_DETECTION` | Detect unexpected prompt cache breaks | API service |
| `QUICK_SEARCH` | Quick search UI shortcut | Typeahead / search |
| `REACTIVE_COMPACT` | Reactive (triggered) context compaction | Compact service; query.ts |
| `REVIEW_ARTIFACT` | Review artifact output handling | Ultrareview command |
| `RUN_SKILL_GENERATOR` | Run skill generation pipeline | Skill system |
| `SELF_HOSTED_RUNNER` | Self-hosted runner support | Runner environment |
| `SHOT_STATS` | Shot statistics tracking | Stats service |
| `SKILL_IMPROVEMENT` | Skill improvement survey | `useSkillImprovementSurvey` hook |
| `SLOW_OPERATION_LOGGING` | Log operations that exceed time thresholds | Debug logging |
| `SSH_REMOTE` | SSH-based remote session | `useSSHSession` hook |
| `STREAMLINED_OUTPUT` | Streamlined tool output formatting | Tool result rendering |
| `TEAMMEM` | Team memory sharing | TeamMemorySync service |
| `TEMPLATES` | Job classifier / template system | Query engine |
| `TERMINAL_PANEL` | Terminal panel capture tool | `TerminalCaptureTool` |
| `TOKEN_BUDGET` | Token budget tracking per request | Budget tracker in query.ts |
| `TORCH` | Torch feature | `torch` command |
| `TRANSCRIPT_CLASSIFIER` | Classify transcripts for routing | Query engine |
| `TREE_SITTER_BASH` | Tree-sitter-based Bash parsing | Bash security/validation |
| `TREE_SITTER_BASH_SHADOW` | Shadow mode for tree-sitter Bash | Bash security A/B |
| `UDS_INBOX` | Unix Domain Socket inbox for agents | `ListPeersTool`; `peers` command |
| `ULTRAPLAN` | Ultra planning mode | `ultraplan` command (internal) |
| `ULTRATHINK` | Extended thinking budget | Model inference |
| `UNATTENDED_RETRY` | Auto-retry in unattended mode | Error handling |
| `UPLOAD_USER_SETTINGS` | Upload settings to Anthropic API | Settings sync service |
| `VERIFICATION_AGENT` | Verification sub-agent for plan execution | `VerifyPlanExecutionTool` |
| `VOICE_MODE` | Voice input via hold-to-talk | `voice` command; `useVoice`, `useVoiceIntegration` hooks |
| `WEB_BROWSER_TOOL` | Playwright-based browser automation | `WebBrowserTool` |
| `WORKFLOW_SCRIPTS` | Bundled workflow scripts | `WorkflowTool`; `workflows` command |
