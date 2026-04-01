# Configuration

Claude Code configuration comes from two parallel systems that serve different purposes:

- **settings.json** — structured key/value settings that control behavior (models, permissions, hooks, plugins, etc.)
- **CLAUDE.md files** — Markdown instructions that are injected verbatim into the system prompt

Both systems are loaded at startup and remain cached for the life of the session.

---

## settings.json — the merge stack

Settings are read from up to five sources and merged in priority order. Later sources override earlier ones.

| Priority (low → high) | Source | File path |
|---|---|---|
| 1 (lowest) | `userSettings` | `~/.claude/settings.json` |
| 2 | `projectSettings` | `<project>/.claude/settings.json` |
| 3 | `localSettings` | `<project>/.claude/settings.local.json` |
| 4 | `flagSettings` | path from `--settings <file>` CLI flag, or inline settings passed via the SDK |
| 5 (highest) | `policySettings` | managed/enterprise settings (see below) |

Merging uses a deep merge (lodash `mergeWith`). Arrays are concatenated and deduplicated rather than replaced — so allow-lists accumulate across sources.

`policySettings` and `flagSettings` are always included regardless of the `--setting-sources` flag. The flag only restricts which of the first three sources are loaded.

### File locations

```
~/.claude/settings.json              # userSettings  (global, all projects)
<project>/.claude/settings.json      # projectSettings (shared, check into git)
<project>/.claude/settings.local.json # localSettings (gitignored automatically)
```

`localSettings` is automatically added to `.gitignore` when it is first written, making it a safe place for developer-specific overrides that should not be shared.

### Plugin settings base

Before the file sources are merged, any settings contributed by installed plugins are applied as the lowest-priority base. All five file sources override plugin settings.

### Policy settings — enterprise / MDM

`policySettings` aggregates settings managed by an organization. Within this single logical source, a "first source wins" rule applies (unlike the rest of the stack, which accumulates from all sources). The sub-sources are checked in this order:

1. **Remote** — settings fetched from Anthropic's managed-settings API (highest within policy)
2. **MDM** — macOS: system preferences plist (`com.anthropic.claudecode`); Windows: HKLM registry key
3. **File** — `<managed-dir>/managed-settings.json` plus drop-in files in `<managed-dir>/managed-settings.d/*.json` (sorted alphabetically; later files win)
4. **HKCU** — Windows: HKCU registry key (user-writable, lowest within policy)

The managed directory is platform-specific (typically `/etc/claude-code/` on macOS/Linux, `%ProgramData%\AnthropicClaude\` on Windows).

Drop-in files follow the systemd/sudoers convention: `10-otel.json` is merged before `20-security.json`, so different teams can ship independent policy fragments without coordinating edits to a single file.

### Reading settings in code

```typescript
import { getInitialSettings } from 'src/utils/settings/settings.js'
const settings = getInitialSettings()   // merged result of all sources
```

Per-source reads:
```typescript
import { getSettingsForSource } from 'src/utils/settings/settings.js'
const userOnly = getSettingsForSource('userSettings')
```

Settings are cached at session level. The cache is invalidated by the file-change detector, but in general Claude Code must be restarted for changes to take effect.

### JSON Schema

The settings schema is published at:

```
https://json.schemastore.org/claude-code-settings.json
```

Add `"$schema": "https://json.schemastore.org/claude-code-settings.json"` to any `settings.json` for editor auto-complete and validation.

---

## CLAUDE.md files — the memory system

CLAUDE.md files are loaded separately from settings.json and serve a different purpose: they inject natural-language instructions directly into Claude's system prompt. They are not structured settings and do not participate in the merge stack.

### Load order

Files are loaded and presented to the model in this order (files loaded later appear later in the prompt and are weighted more heavily by the model):

1. **Managed memory** — `/etc/claude-code/CLAUDE.md` (or the platform equivalent in the managed directory)
2. **User memory** — `~/.claude/CLAUDE.md`
3. **Project memory** — for each directory from the project root up to `/`, in order from farthest to closest:
   - `CLAUDE.md`
   - `.claude/CLAUDE.md`
   - all `.md` files in `.claude/rules/` (sorted alphabetically)
4. **Local memory** — `CLAUDE.local.md` in any of the same directories (gitignored)

Files closer to the current working directory take highest priority (they appear last in the prompt).

### @include directives

Memory files can include other files using `@` notation:

```
@path/to/file.md           # relative path
@./relative/path.md        # explicit relative
@~/home/path.md            # home-relative
@/absolute/path.md         # absolute
```

Included files are inserted before the including file. Circular references are detected and ignored. Only text file extensions are allowed (binary files are rejected).

### Size limits

Each memory file has a recommended maximum of 40,000 characters. Files exceeding this limit are truncated before being passed to the model.

### Interaction with settings.json

The two systems are independent but complementary:

- **settings.json** controls _behavior_ (which tools are allowed, what hooks run, which plugins are enabled).
- **CLAUDE.md** controls _instructions_ (how Claude should think, project conventions, what to do and not do).

settings.json is never read by the model. CLAUDE.md is never parsed for structured settings.

---

## Environment variables

Environment variables are not a general configuration layer — they are special-case inputs used for specific purposes:

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | API key for authentication |
| `CLAUDE_CONFIG_DIR` | Override for `~/.claude/` config directory |
| `CLAUDE_CODE_USE_COWORK_PLUGINS` | Use `cowork_settings.json` instead of `settings.json` |
| `DISABLE_AUTOUPDATER` | Disable automatic CLI updates |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable analytics and non-critical network traffic |

Environment variables that map to settings keys (set via managed MDM or `applyConfigEnvironmentVariables`) are injected early in startup, before the settings merge runs.

---

## Key settings reference

The following settings are commonly used. For the full schema, consult the JSON Schema URL above.

### Model

```jsonc
{
  "model": "claude-opus-4-5"   // main loop model
}
```

### Permissions

```jsonc
{
  "permissions": {
    "defaultMode": "default",          // permission mode (see permissions.md)
    "allow": ["Bash(git *)", "Read"],  // always-allow rules
    "deny": ["Bash(rm -rf *)"],        // always-deny rules
    "additionalDirectories": ["/tmp"]  // extend working directory scope
  }
}
```

### Hooks

```jsonc
{
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "hooks": [{ "type": "command", "command": "echo pre-bash" }] }],
    "PostToolUse": [...],
    "Notification": [...],
    "UserPromptSubmit": [...],
    "SessionStart": [...],
    "SessionEnd": [...],
    "Stop": [...],
    "SubagentStop": [...],
    "PreCompact": [...],
    "PostCompact": [...]
  }
}
```

### Plugins

```jsonc
{
  "enabledPlugins": {
    "my-plugin@my-marketplace": true,
    "other-plugin@builtin": false
  }
}
```

### Cleanup

```jsonc
{
  "cleanupPeriodDays": 30   // how long to retain session transcripts
}
```
