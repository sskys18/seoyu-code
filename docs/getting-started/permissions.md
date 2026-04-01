# Permissions

Claude Code runs tools on your machine, so it has a layered permission system that lets you control exactly what Claude is allowed to do — from full automation to step-by-step confirmation of every action.

---

## Permission modes

A **permission mode** is the global policy that applies when no specific allow/deny rule matches a tool call.

| Mode | What happens when a tool tries to run |
|---|---|
| `default` | Claude asks for confirmation before any destructive or side-effectful action (file writes, Bash commands, etc.). Read-only tools are allowed automatically. |
| `acceptEdits` | File edits and creations are auto-approved without prompting. Bash commands still require confirmation. |
| `plan` | No tools that modify state are executed. Claude produces a plan/analysis only. |
| `bypassPermissions` | All tool calls are auto-approved without prompting. **Dangerous** — requires explicit opt-in via a dialog. |
| `dontAsk` | Claude never prompts for permission; denied tools fail silently with an error result instead of asking the user. Intended for non-interactive/headless use. |

### The `auto` mode (internal only)

A sixth mode, `auto`, exists but is only active when the `TRANSCRIPT_CLASSIFIER` feature flag is enabled. This flag is only present in internal Anthropic builds and is not available in public releases. In `auto` mode, a classifier model evaluates each tool call and decides whether to allow or deny it automatically, based on the conversation context. Do not reference `auto` in user-facing documentation for public builds.

### How modes interact with plan mode

When entering `plan` mode, Claude Code stores the previous mode in `prePlanMode` so it can be restored when plan mode ends.

---

## Setting the permission mode

### CLI flag (per-session)

```bash
claude --permission-mode acceptEdits
claude -p "fix the build" --permission-mode bypassPermissions
```

The `--permission-mode` flag accepts any of the five external modes listed above.

### settings.json (persistent)

```jsonc
// ~/.claude/settings.json  (user-global)
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}

// <project>/.claude/settings.json  (per-project)
{
  "permissions": {
    "defaultMode": "default"
  }
}
```

Project settings override user settings; policy settings override both (see [configuration.md](./configuration.md) for the full merge order).

### Per-command (in the REPL)

Use `/permissions` in the REPL to view and change the current mode interactively.

---

## Permission rules

In addition to the global mode, you can define fine-grained **rules** that always allow, always deny, or always ask for specific tools or tool inputs.

### Rule syntax

A rule is a string of the form:

```
ToolName
ToolName(content pattern)
```

- `ToolName` alone matches any use of that tool.
- `ToolName(pattern)` matches only when the tool input matches `pattern`. Patterns support glob syntax for Bash commands (`Bash(git *)`), file paths (`Read(/tmp/*)`, `Write(src/**)`), and similar.

### Rule sources

Rules can come from any settings source. Rules from all active sources are combined — a tool call is denied if any deny rule matches, allowed if any allow rule matches (and no deny rule matches), and the mode default applies otherwise.

| Source | How to set |
|---|---|
| `userSettings` | `~/.claude/settings.json` |
| `projectSettings` | `<project>/.claude/settings.json` |
| `localSettings` | `<project>/.claude/settings.local.json` |
| `flagSettings` | `--settings <file>` CLI flag |
| `policySettings` | Managed/enterprise settings |
| `cliArg` | `--allow-tool`, `--deny-tool` CLI flags |
| `command` | Rules defined inside a slash command or skill definition |
| `session` | Rules added interactively during a session (not persisted) |

### Example rules in settings.json

```jsonc
{
  "permissions": {
    "allow": [
      "Bash(git *)",          // allow all git subcommands
      "Bash(npm run *)",      // allow npm scripts
      "Read",                 // allow all file reads
      "Write(src/**)"         // allow writes inside src/
    ],
    "deny": [
      "Bash(rm -rf *)",       // never run rm -rf
      "Bash(curl * | bash)"   // never pipe curl to bash
    ],
    "ask": [
      "Bash(sudo *)"          // always ask before sudo
    ]
  }
}
```

### Additional working directories

By default Claude can only read and write within the current working directory (the directory you launched Claude Code from). You can expand this scope:

```jsonc
{
  "permissions": {
    "additionalDirectories": [
      "/tmp",
      "/Users/you/shared-libs"
    ]
  }
}
```

Directories can also be added per-session via the `/permissions` command.

---

## Tool-specific permission behavior

Each tool has its own logic for what requires confirmation:

### Bash (`Bash`)

The most sensitive tool. Any command that modifies state (writes files, changes system configuration, network calls, etc.) requires confirmation in `default` mode. Commands flagged as dangerous by the safety checker (e.g., `rm -rf /`, commands that write to shell configs, Windows path-bypass patterns) are always denied regardless of mode.

### File read tools (`Read`, `Glob`, `Grep`)

Read-only tools are auto-allowed in `default` mode unless an explicit `deny` rule matches.

### File write/edit tools (`Write`, `Edit`, `MultiEdit`)

Require confirmation in `default` mode. Auto-allowed in `acceptEdits` and `bypassPermissions` modes. Writes outside the working directory (and any declared additional directories) are blocked.

### Sub-agent tools (`Task`, `Agent`)

Sub-agents inherit the permission context of the parent session. The parent's mode and rules are passed down.

### MCP tools

MCP tools (from connected MCP servers) go through the same permission pipeline as built-in tools. You can write allow/deny rules using the MCP tool name (e.g., `mcp__myserver__my_tool`).

---

## How a permission decision is made

When the model calls a tool, `useCanUseTool` runs the following pipeline:

1. **`hasPermissionsToUseTool`** — evaluates mode + rules to produce an initial `allow`, `ask`, or `deny` decision.
2. **If `allow`** — the tool executes immediately.
3. **If `deny`** — the tool call is rejected. In `dontAsk` mode the rejection is returned silently; otherwise a denial message is shown.
4. **If `ask`** — the user is prompted. The prompt may include auto-approval suggestions (rules to add to settings). Swarm workers escalate to their coordinator instead of prompting the user directly.

The reason for each decision is recorded as a `PermissionDecisionReason` and included in analytics events for auditing.

### `disableBypassPermissionsMode`

Enterprise policy can set `permissions.disableBypassPermissionsMode: true` in managed settings to prevent users from ever entering `bypassPermissions` mode, even if they have accepted the bypass dialog.
