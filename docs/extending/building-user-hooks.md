# Building User Hooks

Hooks let you inject custom behavior into Claude Code's execution lifecycle. They run at defined events (e.g., before a tool call, after a session starts) and can block, modify, or observe what the model does.

## Defining Hooks in settings.json

Hooks live under the `hooks` key in any Claude Code settings file:

- `~/.claude/settings.json` — user scope (all projects)
- `.claude/settings.json` — project scope
- `.claude/settings.local.json` — local override (not committed)
- Plugin `hooks/hooks.json` — contributed by a plugin

The structure is:

```json
{
  "hooks": {
    "<HookEvent>": [
      {
        "matcher": "<glob-or-regex>",
        "hooks": [
          { "type": "command", "command": "echo hello" }
        ]
      }
    ]
  }
}
```

Each entry under a `HookEvent` is a **matcher group** — a `matcher` pattern plus one or more hook commands. All hooks in a matching group run when the event fires and the matcher matches.

## Hook Events

| Event | Fired when |
|-------|-----------|
| `PreToolUse` | Before any tool call executes |
| `PostToolUse` | After a tool call completes |
| `UserPromptSubmit` | After the user submits a prompt, before the model sees it |
| `Stop` | When the model finishes a turn |
| `SubagentStop` | When a subagent (forked agent) finishes |
| `Notification` | When Claude Code sends a desktop notification |
| `SessionStart` | Once, when a session is initialized |
| `Setup` | During first-time setup |
| `PreSkillExecution` | Before a skill's prompt is sent to the model |
| `PostSkillExecution` | After a skill completes |

`SessionStart` and `Setup` are always emitted in SDK mode; all other events require `includeHookEvents: true` in SDK options.

## Matcher Patterns

The `matcher` field is matched against the tool name (for `PreToolUse`/`PostToolUse`) or is unused (for session-level events). Patterns support glob-style wildcards:

```json
{ "matcher": "Bash" }           // exact tool name
{ "matcher": "Bash(*)" }        // Bash with any command
{ "matcher": "Bash(git *)" }    // Bash with git commands only
{ "matcher": "*" }              // any tool
```

An empty or absent `matcher` matches all events of that type.

## Hook Types

### Command Hooks

Runs a shell command. The hook receives JSON on stdin describing the event context (tool name, inputs, etc.) and can write JSON to stdout to influence behavior.

```json
{
  "type": "command",
  "command": "my-linter --check $FILE",
  "shell": "bash",
  "timeout": 30,
  "if": "Bash(git commit *)"
}
```

Fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `command` | string | required | Shell command to execute |
| `shell` | string | `bash` | Shell interpreter (`bash`, `sh`, `zsh`, `pwsh`) |
| `timeout` | number | 60 | Timeout in seconds |
| `if` | string | — | Conditional: only run if this tool pattern matches |

The command's exit code signals the outcome:
- `0` — success, continue normally
- `2` — block the tool call (only for `PreToolUse`)
- Other non-zero — error, logged but execution continues

### HTTP Hooks

POSTs a JSON body to an HTTP endpoint. Useful for webhooks and policy servers.

```json
{
  "type": "http",
  "url": "https://my-policy-server.example.com/hooks",
  "timeout": 10
}
```

The request body is the same JSON payload that command hooks receive on stdin. The response body (if valid JSON with `{ "decision": "block", "reason": "..." }`) can block tool execution for `PreToolUse`.

HTTP hooks are subject to the SSRF guard (see below).

### Prompt Hooks

Runs a sub-model call with a templated prompt. The `$ARGUMENTS` placeholder in the prompt receives the JSON event payload.

```json
{
  "type": "prompt",
  "prompt": "Summarize the following tool use: $ARGUMENTS",
  "timeout": 30
}
```

Prompt hooks are billed against the active session's API key.

### Agent Hooks

Runs a full multi-turn agentic sub-call. The agent has access to the same tools as the parent (minus tools disallowed for subagents) and must respond via a `StructuredOutput` tool with the hook result schema.

```json
{
  "type": "agent",
  "prompt": "Review this bash command for safety: $ARGUMENTS",
  "timeout": 60
}
```

Agent hooks are powerful but slow — use them for complex policy decisions that require tool access (e.g., checking a command against a risk database).

## Hook Execution Lifecycle

For each fired event:

1. All matching hook groups (across all settings scopes) are collected. Priority order: `localSettings` > `projectSettings` > `userSettings` > `pluginHook` > `builtinHook`.
2. Within each group, hooks execute sequentially.
3. For `PreToolUse`, if any hook returns a block decision, the tool call is cancelled and the model receives an error message.
4. For other events, hook results are informational — they may inject text into the session but cannot cancel the action.
5. The `timeout` field on each hook is a hard deadline; hooks that exceed it are killed and their result is treated as an error.

## Session Hooks (In-Memory)

Session hooks are registered programmatically for the duration of a single session. They do not persist to disk. The SDK `initialize` control request supports registering session hook callbacks:

```typescript
// In SDK initialization
{
  hooks: {
    PreToolUse: [{
      matcher: 'Bash',
      hookCallbackIds: ['my-callback-id'],
      timeout: 30
    }]
  }
}
```

Session hooks have the lowest priority and are labelled `sessionHook` in the UI.

## SSRF Guard

HTTP hooks go through an SSRF guard that blocks connections to private and link-local address ranges:

| Blocked range | Reason |
|--------------|--------|
| `10.0.0.0/8` | Private |
| `172.16.0.0/12` | Private |
| `192.168.0.0/16` | Private |
| `169.254.0.0/16` | Link-local (cloud metadata endpoints) |
| `100.64.0.0/10` | CGNAT / some cloud metadata |
| `0.0.0.0/8` | "This" network |
| `fc00::/7` | IPv6 unique local |
| `fe80::/10` | IPv6 link-local |

**Loopback is intentionally allowed** (`127.0.0.0/8`, `::1`) — local policy servers running on `localhost` are a primary HTTP hook use case.

The guard performs DNS resolution at connection time and validates the resolved IP, preventing DNS rebinding attacks. Both IPv4 and IPv4-mapped IPv6 addresses are checked.

To route HTTP hooks through a proxy (which performs its own DNS resolution and may bypass this guard), configure the global proxy or the sandbox network proxy.

## Conditional Execution (`if`)

The `if` field on a command hook restricts execution to specific tool patterns using the same glob syntax as `matcher`:

```json
{
  "type": "command",
  "command": "security-check.sh",
  "if": "Bash(curl *)"
}
```

This hook fires only when the matched event involves `Bash` commands starting with `curl`, even if the containing matcher group matches all `Bash` commands.

Two hooks with the same `command` but different `if` conditions are treated as distinct hooks and both execute independently.

## Policy Controls

Administrators can set `allowManagedHooksOnly: true` in policy settings to prevent user/project/local hooks from running. When set:
- The hook list UI is empty.
- Only managed hooks (from the policy settings file) execute.
- Plugin hooks are also suppressed.

## Example: Blocking Dangerous Commands

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash(rm -rf *)",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"decision\":\"block\",\"reason\":\"rm -rf is not allowed\"}'"
          }
        ]
      }
    ]
  }
}
```

## Example: Post-Edit Linting

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "eslint --fix $(jq -r .tool_input.path < /dev/stdin) 2>&1 || true",
            "timeout": 15
          }
        ]
      }
    ]
  }
}
```

## Example: Session Start Notification

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "notify-send 'Claude Code session started'"
          }
        ]
      }
    ]
  }
}
```
