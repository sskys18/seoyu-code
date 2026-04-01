# Feature Flags

## 1. Purpose

Feature flags control which capabilities, tools, commands, and behaviors are compiled into and enabled at runtime in Claude Code. Two mechanisms work together: Bun's `bun:bundle` DCE (dead code elimination) for build-time binary flags that produce zero-cost no-ops in disabled builds, and GrowthBook for runtime flag evaluation that can be toggled per-user or per-session without a rebuild.

## 2. Key Files

| File | Size | Role |
|---|---|---|
| `src/services/analytics/growthbook.ts` | 39.6 KB | GrowthBook client initialization, `getGrowthBookFeatureValue()`, `isFeatureEnabled()` |
| `src/services/analytics/index.ts` | 5.4 KB | Analytics event logging, exports `logEvent` |
| `src/services/analytics/firstPartyEventLogger.ts` | 14.2 KB | Ant-internal event logger |
| `src/services/analytics/metadata.ts` | 31.9 KB | Event metadata builders; reads several flags to enrich analytics payloads |
| `src/services/analytics/datadog.ts` | 8.9 KB | Datadog sink for metrics |
| `src/tools.ts` | ~390 lines | Primary site of `feature()` calls for tool registration |
| `src/commands.ts` | ~755 lines | Primary site of `feature()` calls for command registration |

## 3. Bun DCE Mechanism

The `feature()` function is imported from the virtual module `bun:bundle`:

```typescript
import { feature } from 'bun:bundle'
```

During the Bun build, the bundler evaluates `feature('FLAG_NAME')` as a boolean constant. When the flag is `false` (disabled), the entire branch is dead code and eliminated from the bundle — the tool or command has zero runtime footprint. When `true`, the code is inlined and the `require()` call is resolved normally.

**Pattern used throughout the codebase:**

```typescript
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const WorkflowTool = feature('WORKFLOW_SCRIPTS')
  ? (() => {
      require('./tools/WorkflowTool/bundled/index.js').initBundledWorkflows()
      return require('./tools/WorkflowTool/WorkflowTool.js').WorkflowTool
    })()
  : null
```

## 4. All Flags

The following flags appear in the source via `feature('...')` calls. Flags are grouped by their primary domain.

### Agent Behavior Flags

| Flag | What it gates |
|---|---|
| `PROACTIVE` | `SleepTool`, `proactive` command, cron scheduler startup, background scheduling logic |
| `KAIROS` | Proactive scheduling, `assistantCommand`, `briefCommand`, `SendUserFileTool`, `PushNotificationTool`, channel permission callbacks, Dream bundled skill |
| `KAIROS_DREAM` | Dream memory-consolidation bundled skill |
| `KAIROS_BRIEF` | `briefCommand` (standalone, without full KAIROS) |
| `KAIROS_CHANNELS` | Channel-based permission callbacks |
| `KAIROS_GITHUB_WEBHOOKS` | `SubscribePRTool`, `subscribePr` command |
| `KAIROS_PUSH_NOTIFICATION` | `PushNotificationTool` (standalone) |
| `AGENT_TRIGGERS` | `CronCreateTool`, `CronDeleteTool`, `CronListTool`, cron scheduler in `print.ts`, AGENT_TRIGGERS bundled skill |
| `AGENT_TRIGGERS_REMOTE` | `RemoteTriggerTool`, remote triggers bundled skill |
| `MONITOR_TOOL` | `MonitorTool` (tool), `MonitorMcpTask` (task), Bash prompt monitor hints |
| `WORKFLOW_SCRIPTS` | `WorkflowTool`, `LocalWorkflowTask`, `workflowsCmd`, workflow commands |
| `COORDINATOR_MODE` | Coordinator mode check, `TaskStopTool`/`SendMessageTool` in simple mode, coordinator permission handler |

### Multi-Agent / Swarm Flags

| Flag | What it gates |
|---|---|
| `FORK_SUBAGENT` | `forkCmd` command |
| `BUDDY` | `buddy` command |
| `UDS_INBOX` | `ListPeersTool`, `peersCmd` command |

### Permission / Security Flags

| Flag | What it gates |
|---|---|
| `TRANSCRIPT_CLASSIFIER` | `auto` permission mode availability, classifier-based auto-approval/denial in `useCanUseTool`, `YoloClassifierResult` path |
| `BASH_CLASSIFIER` | Speculative bash allow-classifier check, `pendingClassifierCheck` plumbing, shadow parse path in bash permissions |
| `TREE_SITTER_BASH` | Tree-sitter-based bash command parsing for permission matching |
| `TREE_SITTER_BASH_SHADOW` | Shadow mode: runs tree-sitter parser alongside legacy parser for comparison/validation |

### UI / Experience Flags

| Flag | What it gates |
|---|---|
| `VOICE_MODE` | `voiceCommand`, voice streaming STT service |
| `BRIDGE_MODE` | `bridge` command, bridge-mode permission callbacks, `BriefTool` upload path |
| `DAEMON` | `remoteControlServerCommand` (requires BRIDGE_MODE too) |
| `HISTORY_SNIP` | `SnipTool`, `forceSnip` command, snip module in query/QueryEngine |
| `TERMINAL_PANEL` | `TerminalCaptureTool` |
| `CONTEXT_COLLAPSE` | Context collapse logic in query.ts and QueryEngine.ts, `CtxInspectTool` |
| `OVERFLOW_TEST_TOOL` | `OverflowTestTool` (testing only) |
| `WEB_BROWSER_TOOL` | `WebBrowserTool` |
| `CCR_REMOTE_SETUP` | `webCmd` (remote setup command) |
| `HISTORY_PICKER` | History picker UI |
| `MESSAGE_ACTIONS` | Message action affordances |
| `QUICK_SEARCH` | Quick search UI |
| `STREAMLINED_OUTPUT` | Streamlined output rendering in `print.ts` |
| `AUTO_THEME` | Automatic terminal theme detection |

### Memory / Knowledge Flags

| Flag | What it gates |
|---|---|
| `EXTRACT_MEMORIES` | Memory extraction after sessions, `extractMemoriesModule` in stop hooks and print.ts |
| `TEAMMEM` | Team memory paths, team-shared memory writes in `extractMemories.ts` |
| `AGENT_MEMORY_SNAPSHOT` | Agent memory snapshot functionality |

### Context / Compaction Flags

| Flag | What it gates |
|---|---|
| `REACTIVE_COMPACT` | Reactive compaction trigger in query.ts |
| `CACHED_MICROCOMPACT` | Cached micro-compaction (cache-friendly compaction path) |
| `COMPACTION_REMINDERS` | Compaction reminder notifications |
| `TOKEN_BUDGET` | Token budget tracker in query.ts |
| `BG_SESSIONS` | Background session task summary module |
| `FILE_PERSISTENCE` | File persistence timing in print.ts |

### Skills / Templates Flags

| Flag | What it gates |
|---|---|
| `EXPERIMENTAL_SKILL_SEARCH` | Local skill search index, `clearSkillIndexCache` |
| `MCP_SKILLS` | MCP-provided skill commands (`getMcpSkillCommands`) |
| `TEMPLATES` | Job classifier for template matching in stop hooks and query.ts |
| `RUN_SKILL_GENERATOR` | Skill generator bundled skill |
| `REVIEW_ARTIFACT` | Review artifact bundled skill |
| `BUILDING_CLAUDE_APPS` | Building Claude Apps bundled skill |
| `SKILL_IMPROVEMENT` | Skill improvement feedback loop |

### Infrastructure / Platform Flags

| Flag | What it gates |
|---|---|
| `CHICAGO_MCP` | Chicago MCP integration in query.ts and metadata.ts |
| `DOWNLOAD_USER_SETTINGS` | Remote settings download on session start |
| `UPLOAD_USER_SETTINGS` | Remote settings upload |
| `COMMIT_ATTRIBUTION` | Commit attribution tracking in print.ts |
| `ULTRAPLAN` | `ultraplan` command |
| `TORCH` | `torch` command |
| `ULTRATHINK` | Ultra-thinking extended reasoning mode |
| `COWORKER_TYPE_TELEMETRY` | Coworker type field in analytics metadata |
| `NATIVE_CLIENT_ATTESTATION` | Native client attestation for auth |
| `NATIVE_CLIPBOARD_IMAGE` | Native clipboard image support |
| `SELF_HOSTED_RUNNER` | Self-hosted runner configuration |
| `BYOC_ENVIRONMENT_RUNNER` | Bring-Your-Own-Cloud environment runner |
| `SSH_REMOTE` | SSH remote agent support |
| `DIRECT_CONNECT` | Direct connection mode |
| `CCR_AUTO_CONNECT` | Auto-connect for CCR |
| `CCR_MIRROR` | CCR mirror mode |
| `BREAK_CACHE_COMMAND` | Break-cache command visibility |
| `ABLATION_BASELINE` | Ablation baseline telemetry |
| `ENHANCED_TELEMETRY_BETA` | Enhanced telemetry beta features |
| `SHOT_STATS` | Shot statistics tracking |
| `PERFETTO_TRACING` | Perfetto trace integration |
| `SLOW_OPERATION_LOGGING` | Log slow operation warnings |
| `HARD_FAIL` | Hard-fail on certain error conditions (vs. graceful degradation) |
| `UNATTENDED_RETRY` | Auto-retry for unattended/background agents |
| `DUMP_SYSTEM_PROMPT` | Debug: dump system prompt to disk |
| `ALLOW_TEST_VERSIONS` | Allow test model versions |
| `HOOK_PROMPTS` | Hook prompts for pre/post tool events |
| `PROMPT_CACHE_BREAK_DETECTION` | Detect prompt cache breaks |
| `MEMORY_SHAPE_TELEMETRY` | Memory shape telemetry |
| `AWAY_SUMMARY` | Away-mode session summary |
| `IS_LIBC_GLIBC` | Runtime: target is glibc Linux |
| `IS_LIBC_MUSL` | Runtime: target is musl Linux (Alpine) |
| `NEW_INIT` | New init flow |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Built-in explore/plan agent definitions |
| `LODESTONE` | Lodestone integration |
| `CONNECTOR_TEXT` | Connector text in UI |
| `ANTI_DISTILLATION_CC` | Anti-distillation prompt injection |
| `POWERSHELL_AUTO_MODE` | PowerShell auto-mode for Windows |
| `VERIFICATION_AGENT` | Verification agent for plan execution |
| `MCP_RICH_OUTPUT` | Rich output support for MCP tools |

## 5. GrowthBook Integration

GrowthBook is the runtime feature-flag backend. It evaluates flags against user attributes (user ID, account type, region, etc.) allowing gradual rollouts, A/B tests, and per-user overrides without requiring a new build.

Key integration points in `src/services/analytics/growthbook.ts`:
- Client is initialized once on startup with user identity attributes
- `isFeatureEnabled(flagName)` — boolean check
- `getGrowthBookFeatureValue(flagName, defaultValue)` — typed value fetch
- Flags in GrowthBook can override or supplement the build-time `feature()` values for flags that are not DCE-gated

## 6. Design Decisions

- **DCE over runtime `process.env`**: Using `feature()` from `bun:bundle` means disabled features are entirely absent from production binaries, reducing attack surface and bundle size. A `process.env.USER_TYPE === 'ant'` guard is used for ant-employee-only features that need to be present in the binary but gated at runtime (e.g., `REPLTool`, `ConfigTool`, `TungstenTool`).
- **Feature-flag composition**: Many capabilities require multiple flags (e.g., `remoteControlServerCommand` needs both `DAEMON` and `BRIDGE_MODE`). This allows fine-grained rollout of complex features by enabling prerequisites independently.
- **No top-level `process.env` reads**: A custom ESLint rule (`no-process-env-top-level`) prevents reading `process.env` at module initialization time (DCE boundary), which would prevent Bun from proving the branch dead. `eslint-disable` comments are required when necessary and kept minimal.
- **Flags are convention, not enforcement**: The `feature()` API is a thin build-time constant; there is no central registry of what flags exist. The exhaustive list above was derived by grepping all `feature('...')` calls in the source.
