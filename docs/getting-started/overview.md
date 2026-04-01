# Overview

Claude Code is Anthropic's official CLI for Claude. It lets you work with Claude directly from your terminal — asking questions, editing files, running shell commands, and completing multi-step engineering tasks, all through a conversational interface.

The codebase is a TypeScript monolith (1,884 files, 512K+ LOC) built with Bun, rendered in the terminal via React and Ink.

---

## What Claude Code does

At the most basic level, Claude Code accepts a prompt, sends it to the Claude API along with context about your project, and streams back a response. What makes it more than a chat client is the **tool system**: Claude can call tools to read files, write code, run shell commands, search the codebase, and more — all within the same conversation. You stay in the loop at every step.

---

## How a user interaction flows

```
User types a prompt
        |
        v
processUserInput        <-- slash commands (/help, /clear, etc.) are handled first
        |
        v
QueryEngine.submitMessage()   <-- one turn in the conversation
        |
        v
fetchSystemPromptParts        <-- assemble system prompt (CLAUDE.md, tool list, context)
        |
        v
query() → Claude API          <-- streamed LLM call with messages + tools
        |
        v
Model returns response
        |
        +--> text block        <-- streamed to the terminal immediately
        |
        +--> tool_use block    <-- one or more tool calls in the same response
                |
                v
        canUseTool()           <-- permission check (see permissions.md)
                |
        [allowed]  [ask]  [deny]
                |
                v
        tool.call(input)       <-- tool executes (file read/write, bash, grep, …)
                |
                v
        tool result appended to messages
                |
                v
        loop back to model ----^   (continues until model emits stop_reason)
        |
        v
Final text response displayed; session persisted to disk
```

The `QueryEngine` class (`src/QueryEngine.ts`) owns the per-conversation state — messages, file-read cache, cumulative token usage, and permission denials. A new `QueryEngine` is created for each conversation; `submitMessage()` starts a new turn within it.

---

## Key concepts

### Tools

Tools are the primitives Claude uses to interact with your system. Each tool is a TypeScript class that declares:

- `name` — the identifier the model uses in `tool_use` blocks
- `description` — what the tool does (shown to the model in the system prompt)
- `inputSchema` — JSON Schema for the input the model must provide
- `call(input)` — the implementation that runs on your machine

Built-in tools include file read/write/edit, Bash execution, ripgrep search, a web fetch tool, and sub-agent spawning. MCP (Model Context Protocol) servers add additional tools at runtime.

### Commands

Commands are slash-commands available in the REPL (`/help`, `/clear`, `/permissions`, etc.). They are distinct from tools — commands are user-invoked and run locally without a model call, while tools are model-invoked and require a round-trip to the API.

**Skills** are a special kind of command: they inject a prompt into the conversation and let the model handle the rest. Skills can be bundled with the CLI, provided by plugins, or loaded from `.claude/commands/` in your project.

### Permissions

Before any tool executes, the permission system (`src/hooks/useCanUseTool.tsx`) decides whether to allow, deny, or ask the user. Decisions are based on the current **permission mode**, configured allow/deny rules, and the specific tool being called. See [permissions.md](./permissions.md) for the full reference.

### Sessions

Each conversation is a **session**. Sessions have a UUID, and their message transcripts are stored on disk (under `~/.claude/projects/`). When you restart Claude Code you can resume a previous session. The `QueryEngine` holds the in-memory message list; `sessionStorage` utilities handle persistence.

### Memory (CLAUDE.md files)

Claude Code loads Markdown files into the system prompt before every conversation. These files — collectively called "memory" — carry project instructions, coding conventions, and tool usage hints. They are loaded in the following order (later files take precedence):

1. **Managed memory** — `/etc/claude-code/CLAUDE.md` (set by IT/MDM)
2. **User memory** — `~/.claude/CLAUDE.md` (your personal global instructions)
3. **Project memory** — `CLAUDE.md`, `.claude/CLAUDE.md`, and `.claude/rules/*.md` files discovered by walking from the current directory up to the filesystem root
4. **Local memory** — `CLAUDE.local.md` files (gitignored, for private per-project notes)

Files closer to the current directory are loaded last and therefore carry more weight. Memory files can include other files with `@path` directives.

The 40,000-character recommended maximum per file is enforced with a soft warning; larger files are truncated before being passed to the model.

---

## Technology stack

| Layer | Technology |
|---|---|
| Runtime | Bun |
| Language | TypeScript (strict) |
| Terminal UI | React + Ink |
| LLM API | Anthropic Messages API (`@anthropic-ai/sdk`) |
| Tool protocol | Model Context Protocol (MCP) for external tools |
| Persistence | JSON files in `~/.claude/` |
