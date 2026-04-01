# Claude Code Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create 38 markdown documentation files + slim README covering the entire Claude Code CLI codebase for three audiences (end users, developers, extension authors).

**Architecture:** Each doc is written by reading the relevant source files and extracting module-level information. Docs are independent — no doc depends on another doc's content. All architecture docs follow a 6-section template (Purpose, Key Files, Data Flow, Core Types, Integration Points, Design Decisions). Reference docs are dense tables extracted from source.

**Tech Stack:** GitHub-flavored Markdown, Mermaid diagrams, relative cross-links.

**Spec:** `docs/superpowers/specs/2026-04-01-documentation-design.md`

**Parallelization note:** Tasks 1-8 are independent batches. Each task produces a self-contained set of docs. All tasks can run in parallel via worktree-isolated agents.

---

## Task 1: README.md + docs/index.md (Foundation)

**Files:**
- Modify: `README.md`
- Create: `docs/index.md`

- [ ] **Step 1: Read existing README.md** to understand current content

- [ ] **Step 2: Replace README.md** with slim ~20 line pointer:
  - One paragraph: what this repo contains (Claude Code CLI source, TypeScript/Bun/React+Ink)
  - Link to `docs/index.md` for full documentation
  - Quick links: Getting Started, Architecture, Extending, Reference

- [ ] **Step 3: Create docs/index.md** landing page:
  - What Claude Code is
  - Mermaid system diagram showing all major subsystems:
    ```
    CLI Entry → Input Processing → QueryEngine → Anthropic API
    QueryEngine → Tool System, Permission System
    State Management ↔ Component System → Custom Ink Renderer
    Services (MCP, LSP, OAuth, Analytics)
    Bridge System ↔ IDE Extensions
    Plugin System → Skills, User Hooks, MCP Servers
    Multi-Agent (Swarm/Teams, Coordinator)
    ```
  - Navigation guide with one-line description per section
  - Source files to read for context: `src/main.tsx` (first 100 lines for bootstrap overview)

- [ ] **Step 4: Commit**
```bash
git add README.md docs/index.md
git commit -m "docs: add slim README and docs landing page with system diagram"
```

---

## Task 2: Getting Started — overview.md + configuration.md + permissions.md + plugins-and-skills.md

**Files:**
- Create: `docs/getting-started/overview.md`
- Create: `docs/getting-started/configuration.md`
- Create: `docs/getting-started/permissions.md`
- Create: `docs/getting-started/plugins-and-skills.md`

- [ ] **Step 1: Create docs/getting-started/ directory**

- [ ] **Step 2: Write overview.md**
  - What Claude Code does at a high level
  - How a user interaction flows: prompt → LLM → tool calls → response
  - Key concepts: tools, commands, permissions, sessions, memory
  - Source files to read: `src/main.tsx` (bootstrap flow), `src/QueryEngine.ts` (query lifecycle)

- [ ] **Step 3: Write configuration.md**
  - Settings merge order: userSettings → projectSettings → localSettings → flagSettings → policySettings
  - CLAUDE.md files as a separate config system from settings.json
  - Environment variables as special-case inputs
  - MDM as enterprise config source
  - Source files to read: `src/utils/settings/settings.ts`, `src/utils/settings/constants.ts`, `src/utils/claudemd.ts`

- [ ] **Step 4: Write permissions.md**
  - Permission modes: default, plan, bypassPermissions, dontAsk, acceptEdits
  - Note `auto` is internal-only (TRANSCRIPT_CLASSIFIER flag)
  - Permission rules, source types, allow/deny/ask
  - Source files to read: `src/types/permissions.ts`, `src/hooks/useCanUseTool.tsx` (first 200 lines for patterns)

- [ ] **Step 5: Write plugins-and-skills.md**
  - What plugins are, how to install
  - Plugin capabilities: skills, hooks, MCP servers, LSP servers
  - What skills are, how to invoke with /skillname
  - Source files to read: `src/services/plugins/` (directory listing + main files), `src/skills/bundledSkills.ts`

- [ ] **Step 6: Commit**
```bash
git add docs/getting-started/
git commit -m "docs: add getting-started section (overview, config, permissions, plugins)"
```

---

## Task 3: Getting Started — commands-reference.md

**Files:**
- Create: `docs/getting-started/commands-reference.md`

- [ ] **Step 1: Read src/commands.ts** to extract all command registrations — names, aliases, descriptions, feature flags

- [ ] **Step 2: Identify INTERNAL_ONLY_COMMANDS** to exclude from public docs

- [ ] **Step 3: Write commands-reference.md**
  - Group commands by category based on actual registry (not assumed categories)
  - Each entry: name, aliases, brief description, feature flag gate (if any)
  - Mark feature-gated commands clearly
  - Source files to read: `src/commands.ts` (full file)

- [ ] **Step 4: Commit**
```bash
git add docs/getting-started/commands-reference.md
git commit -m "docs: add commands reference with all public commands"
```

---

## Task 4: Architecture — overview.md + core-engine.md + input-processing.md + state-management.md + performance.md

**Files:**
- Create: `docs/architecture/overview.md`
- Create: `docs/architecture/core-engine.md`
- Create: `docs/architecture/input-processing.md`
- Create: `docs/architecture/state-management.md`
- Create: `docs/architecture/performance.md`

Each architecture doc follows the 6-section template:
1. Purpose (2-3 sentences)
2. Key Files (paths + line counts)
3. Data Flow (diagram where helpful)
4. Core Types (key interfaces/types)
5. Integration Points (connections to other subsystems)
6. Design Decisions (patterns/trade-offs)

- [ ] **Step 1: Create docs/architecture/ directory**

- [ ] **Step 2: Write overview.md**
  - System-level mermaid diagram
  - Request lifecycle: CLI input → Commander.js → React/Ink → QueryEngine → API → Tool execution → Response rendering
  - Tech stack table
  - Source files: `src/main.tsx` (first 200 lines), `src/QueryEngine.ts` (first 200 lines)

- [ ] **Step 3: Write core-engine.md**
  - Bootstrap: main.tsx, bootstrap/state.ts
  - QueryEngine: LLM orchestration, model selection, streaming
  - System prompt: constants/prompts.ts, systemPromptSections.ts
  - REPL: screens/REPL.tsx
  - Source files: `src/main.tsx`, `src/QueryEngine.ts`, `src/query.ts`, `src/constants/prompts.ts` (first 200 lines), `src/screens/REPL.tsx` (first 100 lines)

- [ ] **Step 4: Write input-processing.md**
  - processUserInput pipeline
  - Slash command parsing and dispatch
  - Input classification: slash commands, bash commands, text prompts
  - Source files: list `src/utils/processUserInput/` directory, read main files (first 200 lines each)

- [ ] **Step 5: Write state-management.md**
  - AppState store, state shape, useAppState hook
  - Context providers (src/context/)
  - Settings migrations (src/migrations/)
  - Source files: `src/state/AppState.tsx` (first 300 lines), list `src/context/`, list `src/migrations/`

- [ ] **Step 6: Write performance.md**
  - Parallel prefetch, lazy loading, streaming, context compression
  - Source files: `src/main.tsx` (prefetch sections), `src/services/compact/` (directory listing)

- [ ] **Step 7: Commit**
```bash
git add docs/architecture/overview.md docs/architecture/core-engine.md docs/architecture/input-processing.md docs/architecture/state-management.md docs/architecture/performance.md
git commit -m "docs: add architecture core docs (overview, engine, input, state, performance)"
```

---

## Task 5: Architecture — tool-system.md + command-system.md + permission-system.md + feature-flags.md + task-system.md

**Files:**
- Create: `docs/architecture/tool-system.md`
- Create: `docs/architecture/command-system.md`
- Create: `docs/architecture/permission-system.md`
- Create: `docs/architecture/feature-flags.md`
- Create: `docs/architecture/task-system.md`

All follow the 6-section template.

- [ ] **Step 1: Write tool-system.md**
  - Tool.ts generic type, 42 tools, registration, ToolUseContext, permission integration
  - Source files: `src/Tool.ts`, `src/tools.ts`, list `src/tools/` directory

- [ ] **Step 2: Write command-system.md**
  - commands.ts registry, dynamic imports, feature flag gating, command lifecycle
  - Source files: `src/commands.ts`

- [ ] **Step 3: Write permission-system.md**
  - PermissionMode, PermissionRule, PermissionDecision types
  - Rule sources, decision flow
  - Source files: `src/types/permissions.ts`, `src/hooks/useCanUseTool.tsx` (first 200 lines)

- [ ] **Step 4: Write feature-flags.md**
  - Bun DCE mechanism, all flags, GrowthBook integration
  - Source files: grep for `feature('` across codebase, `src/services/analytics/` (directory listing)

- [ ] **Step 5: Write task-system.md**
  - 7 task types, background execution
  - Source files: `src/tasks/types.ts`, list `src/tasks/` directory

- [ ] **Step 6: Commit**
```bash
git add docs/architecture/tool-system.md docs/architecture/command-system.md docs/architecture/permission-system.md docs/architecture/feature-flags.md docs/architecture/task-system.md
git commit -m "docs: add architecture docs (tools, commands, permissions, flags, tasks)"
```

---

## Task 6: Architecture — ink-renderer.md + component-system.md + hook-ecosystem.md + user-hooks.md + service-layer.md

**Files:**
- Create: `docs/architecture/ink-renderer.md`
- Create: `docs/architecture/component-system.md`
- Create: `docs/architecture/hook-ecosystem.md`
- Create: `docs/architecture/user-hooks.md`
- Create: `docs/architecture/service-layer.md`

All follow the 6-section template.

- [ ] **Step 1: Write ink-renderer.md**
  - Custom Ink fork: renderer, reconciler, Yoga layout, custom components
  - Source files: list `src/ink/` directory, `src/ink/ink.tsx` (first 200 lines), `src/ink/reconciler.ts` (first 200 lines)

- [ ] **Step 2: Write component-system.md**
  - 389 components, categories, relationship to Ink renderer
  - Source files: list `src/components/` subdirectories

- [ ] **Step 3: Write hook-ecosystem.md**
  - React hooks patterns, top hooks by complexity
  - Source files: list `src/hooks/` directory, read first 100 lines of `useTypeahead.tsx`, `useReplBridge.tsx`

- [ ] **Step 4: Write user-hooks.md**
  - User-configurable hooks: agent, HTTP, prompt, session, SSRF guards
  - Source files: list `src/utils/hooks/` directory, read main files

- [ ] **Step 5: Write service-layer.md**
  - 20+ services overview, categories, initialization, external connections
  - Source files: list `src/services/` subdirectories, read index/main files of each

- [ ] **Step 6: Commit**
```bash
git add docs/architecture/ink-renderer.md docs/architecture/component-system.md docs/architecture/hook-ecosystem.md docs/architecture/user-hooks.md docs/architecture/service-layer.md
git commit -m "docs: add architecture docs (ink renderer, components, hooks, services)"
```

---

## Task 7: Architecture — bridge-system.md + multi-agent.md + plugin-system.md + skill-system.md + computer-use.md

**Files:**
- Create: `docs/architecture/bridge-system.md`
- Create: `docs/architecture/multi-agent.md`
- Create: `docs/architecture/plugin-system.md`
- Create: `docs/architecture/skill-system.md`
- Create: `docs/architecture/computer-use.md`

All follow the 6-section template.

- [ ] **Step 1: Write bridge-system.md**
  - IDE bridge, JWT auth, bidirectional messaging, VS Code/JetBrains
  - Source files: list `src/bridge/` directory, `src/bridge/bridgeMain.ts` (first 200 lines)

- [ ] **Step 2: Write multi-agent.md**
  - Swarm/teams system, InProcessTeammateTask, coordinator mode
  - Source files: list `src/utils/swarm/` directory, `src/utils/swarm/inProcessRunner.ts` (first 200 lines), `src/coordinator/coordinatorMode.ts` (first 200 lines)

- [ ] **Step 3: Write plugin-system.md**
  - Plugin discovery, loading, lifecycle, marketplace
  - Source files: list `src/services/plugins/` directory, read main files

- [ ] **Step 4: Write skill-system.md**
  - Skill registration, execution, SkillTool flow
  - Source files: list `src/skills/` directory, `src/skills/bundledSkills.ts`, `src/skills/loadSkillsDir.ts`

- [ ] **Step 5: Write computer-use.md**
  - Computer use capabilities, MCP server, host adapter
  - Source files: list `src/utils/computerUse/` directory, read main files

- [ ] **Step 6: Commit**
```bash
git add docs/architecture/bridge-system.md docs/architecture/multi-agent.md docs/architecture/plugin-system.md docs/architecture/skill-system.md docs/architecture/computer-use.md
git commit -m "docs: add architecture docs (bridge, multi-agent, plugins, skills, computer-use)"
```

---

## Task 8: Extending Section (all 7 docs)

**Files:**
- Create: `docs/extending/building-plugins.md`
- Create: `docs/extending/building-skills.md`
- Create: `docs/extending/building-user-hooks.md`
- Create: `docs/extending/agent-sdk.md`
- Create: `docs/extending/mcp-integration.md`
- Create: `docs/extending/lsp-integration.md`
- Create: `docs/extending/ide-bridge-protocol.md`

- [ ] **Step 1: Create docs/extending/ directory**

- [ ] **Step 2: Write building-plugins.md**
  - Plugin structure, manifest, registration of skills/hooks/MCP servers, marketplace
  - Source files: `src/services/plugins/` (main files)

- [ ] **Step 3: Write building-skills.md**
  - Skill file format, frontmatter, invocation via SkillTool, best practices
  - Source files: `src/skills/bundledSkills.ts`, `src/skills/loadSkillsDir.ts`, example skills in `src/skills/bundled/`

- [ ] **Step 4: Write building-user-hooks.md**
  - How to define hooks in settings.json, hook types, SSRF guards, security model
  - Source files: `src/utils/hooks/` (main files)

- [ ] **Step 5: Write agent-sdk.md**
  - Agent SDK types, core/control schemas, programmatic usage
  - Source files: `src/entrypoints/sdk/coreSchemas.ts` (first 300 lines), `src/entrypoints/sdk/controlSchemas.ts` (first 200 lines), `src/entrypoints/agentSdkTypes.ts`

- [ ] **Step 6: Write mcp-integration.md**
  - MCP client architecture, server discovery, auth, channels, tool normalization
  - Source files: list `src/services/mcp/` directory, read main files

- [ ] **Step 7: Write lsp-integration.md**
  - LSP client setup, capabilities, how code intelligence feeds into tools
  - Source files: list `src/services/lsp/` directory, read main files

- [ ] **Step 8: Write ide-bridge-protocol.md**
  - Bridge architecture, JWT handshake, message types, building new IDE extensions
  - Source files: `src/bridge/bridgeMessaging.ts`, `src/bridge/jwtUtils.ts`

- [ ] **Step 9: Commit**
```bash
git add docs/extending/
git commit -m "docs: add extending section (plugins, skills, hooks, SDK, MCP, LSP, bridge)"
```

---

## Task 9: Reference Section (all 5 docs)

**Files:**
- Create: `docs/reference/tools.md`
- Create: `docs/reference/commands.md`
- Create: `docs/reference/feature-flags.md`
- Create: `docs/reference/hooks.md`
- Create: `docs/reference/services.md`

- [ ] **Step 1: Create docs/reference/ directory**

- [ ] **Step 2: Write tools.md** — table of all 42 tools
  - Columns: Name | Aliases | Category | Input Schema Summary | Description
  - Source files: `src/tools.ts` (tool registry), `src/Tool.ts` (type defs), list `src/tools/` for all tool directories

- [ ] **Step 3: Write commands.md** — table of all public commands
  - Columns: Name | Aliases | Args | Feature Flag | Description
  - Exclude INTERNAL_ONLY_COMMANDS
  - Source files: `src/commands.ts` (full file)

- [ ] **Step 4: Write feature-flags.md** — table of all feature flags
  - Columns: Flag Name | What It Enables | Gated Commands/Tools
  - Source: grep `feature('` across entire codebase, cross-reference with commands.ts

- [ ] **Step 5: Write hooks.md** — table of all React hooks
  - Columns: Hook Name | File | Purpose | Key Dependencies
  - Source files: list `src/hooks/` directory, read first lines of each file for exports

- [ ] **Step 6: Write services.md** — table of all services
  - Columns: Service | Directory | Purpose | External Systems
  - Source files: list `src/services/` subdirectories, read index files

- [ ] **Step 7: Commit**
```bash
git add docs/reference/
git commit -m "docs: add reference section (tools, commands, flags, hooks, services)"
```
