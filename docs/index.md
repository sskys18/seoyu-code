# Claude Code — Documentation

Claude Code is Anthropic's official CLI for Claude, the AI assistant. It is a TypeScript/Bun application with a terminal UI built on a custom React+Ink renderer. From the terminal, Claude Code can read and edit files, run shell commands, search codebases, manage git workflows, browse the web, communicate with IDE extensions, and orchestrate multi-agent workloads — all driven by natural language.

## Repository Scale

| Metric | Count |
|---|---|
| TypeScript source files | 1,884 |
| Lines of code | 512,000+ |
| Agent tools | 42 |
| Slash commands | 103+ |
| Services | 20+ |

## System Diagram

```mermaid
flowchart TD
    CLI["CLI Entry\n(main.tsx / Commander.js)"]
    Input["Input Processing\n(earlyInput, history, context)"]
    QE["QueryEngine\n(streaming, tool-call loop, retry)"]
    API["Anthropic API\n(claude-3/4 models)"]

    ToolSys["Tool System\n(42 tools)"]
    FileTools["File Tools\n(Read, Write, Edit, Glob, Grep)"]
    BashTool["BashTool"]
    SearchTools["Search Tools\n(Web Fetch, Web Search)"]
    AgentTool["AgentTool\n(sub-agents)"]
    OtherTools["Other Tools\n(LSP, Notebook, Task, Cron…)"]

    Perms["Permission System\n(toolPermission hooks)"]

    State["State Management\n(bootstrap/state, AppState)"]
    Components["Component System\n(~140 Ink components)"]
    Renderer["Custom Ink Renderer\n(src/ink/)"]

    Services["Services"]
    MCP["MCP\n(Model Context Protocol)"]
    LSP["LSP\n(Language Server Protocol)"]
    OAuth["OAuth / Keychain"]
    Analytics["Analytics\n(GrowthBook, OpenTelemetry)"]

    Bridge["Bridge System\n(src/bridge/)"]
    IDE["IDE Extensions\n(VS Code, JetBrains)"]

    Plugins["Plugin System\n(src/plugins/, src/skills/)"]
    Skills["Skills"]
    UserHooks["User Hooks"]
    MCPServers["MCP Servers"]

    MultiAgent["Multi-Agent\n(src/coordinator/)"]
    Swarm["Swarm / Teams"]
    Coordinator["Coordinator"]

    CLI --> Input --> QE
    QE --> API
    QE --> ToolSys
    QE --> Perms
    ToolSys --> FileTools
    ToolSys --> BashTool
    ToolSys --> SearchTools
    ToolSys --> AgentTool
    ToolSys --> OtherTools

    State <--> Components --> Renderer
    QE <--> State

    Services --> MCP
    Services --> LSP
    Services --> OAuth
    Services --> Analytics
    QE --> Services

    Bridge <--> IDE
    QE <--> Bridge

    Plugins --> Skills
    Plugins --> UserHooks
    Plugins --> MCPServers
    QE --> Plugins

    MultiAgent --> Swarm
    MultiAgent --> Coordinator
    AgentTool --> MultiAgent
```

## Sections

| Section | Description |
|---|---|
| [getting-started/](getting-started/) | End users: running commands, configuring settings, managing permissions, and installing plugins |
| [architecture/](architecture/) | Developers: internals of QueryEngine, Tool System, Permission System, State Management, and data flow |
| [extending/](extending/) | Extension authors: writing plugins, skills, MCP servers, LSP integrations, and IDE bridge adapters |
| [reference/](reference/) | Quick-lookup tables for all tools, slash commands, CLI flags, hook types, and service interfaces |
