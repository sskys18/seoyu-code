# Claude Code Documentation Design Spec

**Date:** 2026-04-01
**Status:** Approved

## Goal

Create comprehensive `docs/` folder documentation for the Claude Code CLI source code (1,884 files, 512K+ LOC). Serves three audiences at module-level depth.

## Audiences

1. **End users / power users** — commands, config, permissions, plugins
2. **Developers studying internals** — architecture, subsystems, data flow
3. **Plugin/extension authors** — building plugins, skills, MCP/LSP/IDE integrations

## Structure

```
README.md                             # Slim pointer (~20 lines) → docs/index.md
docs/
├── index.md                          # Landing page, system diagram, navigation
├── getting-started/                  # End users & power users
│   ├── overview.md                   # What Claude Code is, mental model
│   ├── commands-reference.md         # All 103 commands, grouped by category
│   ├── configuration.md             # Settings layering: CLAUDE.md, settings.json, MDM, env vars
│   ├── permissions.md               # Permission modes, allow/deny rules
│   └── plugins-and-skills.md        # Installing plugins, using skills
├── architecture/                     # Developers studying internals
│   ├── overview.md                   # High-level diagram, request lifecycle, tech stack
│   ├── core-engine.md               # main.tsx bootstrap, QueryEngine, query pipeline, streaming
│   ├── tool-system.md               # Tool.ts types, 42 tools, registration, ToolUseContext
│   ├── command-system.md            # commands.ts registry, dynamic imports, feature gating
│   ├── state-management.md          # AppState store, React context, useAppState hook
│   ├── permission-system.md         # Permission modes, rules, decisions, useCanUseTool
│   ├── component-system.md          # React/Ink hierarchy, design-system, message rendering
│   ├── hook-ecosystem.md            # Major hooks, patterns, conventions
│   ├── service-layer.md             # 20+ services: API, MCP, LSP, OAuth, analytics, compact
│   ├── bridge-system.md             # IDE bridge protocol, JWT auth, bidirectional messaging
│   ├── plugin-system.md             # Plugin loading, lifecycle, marketplace, hooks
│   ├── skill-system.md              # Skill registration, execution, types
│   ├── task-system.md               # Task types (LocalShell, Agent, Remote, Dream, etc.)
│   ├── feature-flags.md             # Bun DCE, bun:bundle feature(), GrowthBook, all flags
│   └── performance.md              # Prefetch, lazy loading, streaming, context compression
├── extending/                        # Plugin & extension authors
│   ├── building-plugins.md          # Plugin structure, manifest, loading, hooks, MCP servers
│   ├── building-skills.md           # Skill format, frontmatter, invocation, best practices
│   ├── mcp-integration.md           # MCP client architecture, discovery, auth, channels
│   ├── lsp-integration.md           # LSP client, language server capabilities
│   └── ide-bridge-protocol.md       # Bridge messaging, JWT handshake, message types
└── reference/                        # Quick-lookup tables
    ├── tools.md                     # All 42 tools: name, aliases, schema, description
    ├── commands.md                  # All 103 commands: name, aliases, args, feature flag
    ├── feature-flags.md             # All bun:bundle flags: name, what it enables
    ├── hooks.md                     # All 85 hooks: name, purpose, dependencies
    └── services.md                  # All 20+ services: name, purpose, external systems
```

**Total: 31 docs + 1 README**

## README.md (Replacement)

Replace the current verbose README with ~20 lines:
- One paragraph: what this repo contains (Claude Code CLI source code)
- Link to `docs/index.md`
- Quick links to each docs section (getting-started, architecture, extending, reference)
- No architecture deep-dives, no code examples, no feature flag listings

## docs/index.md — Landing Page

Content:
- What Claude Code is (TypeScript/Bun/React+Ink CLI for Claude)
- Repo stats: 1,884 files, 512K+ LOC, 42 tools, 103 commands, 20+ services
- Mermaid system diagram showing major subsystems and connections:
  - CLI Entry → QueryEngine → Anthropic API
  - QueryEngine → Tool System → (File, Search, Bash, Agent, etc.)
  - QueryEngine → Permission System
  - State Management ↔ Component System (React/Ink)
  - Services (MCP, LSP, OAuth, Analytics)
  - Bridge System ↔ IDE Extensions
  - Plugin System → Skills, Hooks, MCP Servers
- Navigation guide: one-line description per section with links

## Getting Started Section

### overview.md
- What Claude Code does at a high level
- How a user interaction flows: prompt → LLM → tool calls → response
- Key concepts: tools, commands, permissions, sessions, memory

### commands-reference.md
- All 103 commands grouped by category:
  - **Git:** /commit, /diff, /review, etc.
  - **Config:** /config, /settings, /permissions, etc.
  - **Memory:** /memory, /compact, etc.
  - **Tasks:** /tasks, /todo, etc.
  - **Session:** /login, /logout, /resume, /clear, etc.
  - **Debug:** /doctor, /verbose, /debug, etc.
  - **Mode:** /vim, /plan, /voice, /brief, etc.
- Each entry: name, aliases, brief description, flags/args

### configuration.md
- Configuration sources in priority order:
  1. CLI arguments
  2. Session settings
  3. Project CLAUDE.md
  4. User CLAUDE.md (~/.claude/CLAUDE.md)
  5. settings.json (local, project, user)
  6. MDM (managed device management)
  7. Environment variables
- Key settings and what they control
- How overrides work

### permissions.md
- Permission modes: default, plan, bypassPermissions, dontAsk, acceptEdits, auto
- How to set permission mode (CLI flag, settings, per-command)
- Permission rules: source types, allow/deny/ask behaviors
- Tool-specific permissions
- What gets prompted vs auto-allowed

### plugins-and-skills.md
- What plugins are, how to install from marketplace
- Plugin capabilities: skills, hooks, MCP servers, LSP servers
- What skills are, how to invoke with /skillname
- How to enable/disable plugins

## Architecture Section

Each doc follows a consistent template:
1. **Purpose** — What this subsystem does (2-3 sentences)
2. **Key Files** — The most important files with line counts
3. **Data Flow** — How data moves through the subsystem (diagram where helpful)
4. **Core Types** — Key interfaces/types that define the subsystem's contracts
5. **Integration Points** — How it connects to other subsystems
6. **Design Decisions** — Notable patterns or trade-offs

### overview.md
- System-level mermaid diagram
- Request lifecycle: CLI input → Commander.js → React/Ink → QueryEngine → API → Tool execution → Response rendering
- Tech stack table: Bun, TypeScript, React+Ink, Commander.js, Zod v4, ripgrep, MCP SDK, Anthropic SDK, OpenTelemetry

### core-engine.md
- `main.tsx`: Bootstrap sequence (prefetch, Commander setup, Ink renderer)
- `QueryEngine.ts`: LLM API orchestration, model selection, streaming
- `query.ts`: Query pipeline, message construction, response handling
- `REPL.tsx`: Main interactive loop (874.9K — the largest file)

### tool-system.md
- `Tool.ts`: Generic Tool type with Input/Output/Progress type params
- Tool categories: File ops, Search, Execution, Multi-agent, Config, Utilities, Workflows
- `tools.ts`: Registration and factory
- `ToolUseContext`: Runtime context passed to every tool call
- Permission integration: how tools declare permission requirements

### command-system.md
- `commands.ts`: Registry with 103 entries
- Dynamic imports for lazy loading
- Feature flag gating pattern
- Command lifecycle: parse → validate → execute → render

### state-management.md
- `AppState.tsx` (22.9K): Central store
- State shape: messages, permissions, model, settings, suggestions
- `useAppState` hook with Object.is memoization
- Providers: AppStateProvider, MailboxProvider, VoiceProvider

### permission-system.md
- `types/permissions.ts`: PermissionMode, PermissionRule, PermissionDecision
- Rule sources: userSettings, projectSettings, localSettings, flagSettings, policySettings, cliArg, command, session
- Decision flow: rule matching → allow/ask/deny → user prompt → passthrough
- `useCanUseTool` hook (39.3K)

### component-system.md
- 389 components across 32 subdirectories
- Major categories: design-system, permissions, messages, settings, MCP, shell, teams, tasks, diff, skills
- React/Ink patterns: Box, Text, terminal rendering

### hook-ecosystem.md
- 85 hooks across 104 files
- Top hooks by complexity: useTypeahead (207.6K), useReplBridge (112.9K), useVoiceIntegration (97.1K)
- Patterns: state hooks, side-effect hooks, bridge hooks, permission hooks

### service-layer.md
- Service categories: API, MCP (23 files), OAuth, LSP, Analytics/GrowthBook, Compact, Plugins, PolicyLimits, RemoteManagedSettings, ExtractMemories
- How services are initialized and accessed
- External system connections

### bridge-system.md
- 31 files in bridge/
- `bridgeMain.ts` (112.9K): Main bridge loop
- JWT authentication flow
- Bidirectional messaging protocol
- IDE support: VS Code, JetBrains
- Capacity and wake management

### plugin-system.md
- Plugin discovery and loading from marketplace
- Plugin capabilities: skills, hooks, MCP servers, LSP servers
- Plugin lifecycle: install → enable → reload → execute

### skill-system.md
- 20 files in skills/
- Skill file format: frontmatter (name, description, type) + body
- Rigid vs flexible skill types
- SkillTool execution flow

### task-system.md
- `Task.ts`: 6 task types
- LocalShellTask, LocalAgentTask: local execution
- RemoteAgentTask: remote coordination
- DreamTask: background processing
- Feature-gated: LocalWorkflowTask (WORKFLOW_SCRIPTS), MonitorMcpTask (MONITOR_TOOL)

### feature-flags.md
- Bun DCE mechanism: `import { feature } from 'bun:bundle'`
- All flags: PROACTIVE, KAIROS, BRIDGE_MODE, DAEMON, VOICE_MODE, AGENT_TRIGGERS, MONITOR_TOOL, WORKFLOW_SCRIPTS, HISTORY_SNIP, COORDINATOR_MODE, TRANSCRIPT_CLASSIFIER
- GrowthBook integration for runtime flags
- What each flag gates (commands, tools, UI features)

### performance.md
- Parallel prefetch: MDM + keychain at startup
- Lazy loading: heavy modules (OpenTelemetry, gRPC) loaded on demand
- Streaming responses: incremental rendering
- Context compression: compact command, snipping service

## Extending Section

### building-plugins.md
- Plugin directory structure and manifest
- How to register skills, hooks, MCP servers
- Plugin marketplace: how to publish
- Example plugin walkthrough

### building-skills.md
- Skill file format with frontmatter
- Skill invocation via SkillTool
- Rigid vs flexible patterns
- Best practices for skill authoring

### mcp-integration.md
- MCP client in services/mcp/ (23 files)
- Server discovery and connection
- Auth flows for MCP servers
- Channel types and tool normalization
- How to expose tools via MCP

### lsp-integration.md
- LSP client setup
- Language server capabilities leveraged
- How code intelligence feeds into tools

### ide-bridge-protocol.md
- Bridge architecture: CLI ↔ IDE extension
- JWT handshake sequence
- Message types and protocol
- How to build a new IDE extension

## Reference Section

All reference docs are dense tables with minimal prose.

### tools.md
Table columns: Name | Aliases | Category | Input Schema Summary | Description

### commands.md
Table columns: Name | Aliases | Args | Feature Flag | Description

### feature-flags.md
Table columns: Flag Name | What It Enables | Gated Commands/Tools

### hooks.md
Table columns: Hook Name | File | Purpose | Key Dependencies

### services.md
Table columns: Service | Directory | Purpose | External Systems

## Implementation Notes

- All docs written in GitHub-flavored Markdown
- Diagrams use Mermaid syntax (renders natively on GitHub)
- Cross-references use relative links between docs
- Each architecture doc follows the consistent 6-section template
- Reference tables are generated by reading the actual source code to ensure accuracy
