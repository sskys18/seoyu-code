# Plugins and Skills

Claude Code is extensible through two related mechanisms:

- **Plugins** — packages that bundle one or more capabilities (skills, hooks, MCP servers, LSP servers, agents, slash commands).
- **Skills** — slash commands that inject a prompt into the conversation and let Claude handle the rest.

Skills are the user-facing interaction model. Plugins are the delivery mechanism for distributing skills and other capabilities.

---

## Plugins

### What a plugin is

A plugin is a directory (typically a git repository) containing a manifest file (`claude-plugin.json` or `claude-plugin.yaml`) that describes what the plugin provides. A plugin can contribute any combination of:

| Capability | What it does |
|---|---|
| **Skills** | Slash commands (`/skillname`) that invoke a stored prompt |
| **Hooks** | Scripts that run at defined lifecycle points (pre/post tool use, session start/end, etc.) |
| **MCP servers** | Additional tool servers connected via the Model Context Protocol |
| **LSP servers** | Language server protocol servers for editor-style code intelligence |
| **Agents** | Named agent definitions for sub-agent tasks |
| **Slash commands** | Richer commands with their own prompt files |
| **Output styles** | Custom display renderers |

### Plugin scopes

Plugins are installed at one of four scopes:

| Scope | Settings file | Applies to |
|---|---|---|
| `user` | `~/.claude/settings.json` | All projects for this user |
| `project` | `<project>/.claude/settings.json` | This project (shared; check into git) |
| `local` | `<project>/.claude/settings.local.json` | This project (gitignored; private) |
| `managed` | Managed settings (admin-controlled) | Enforced by IT/MDM; cannot be uninstalled by users |

### Installing from a marketplace

Plugins are distributed through **marketplaces** — git repositories that list available plugins. The default Anthropic marketplace is pre-configured; organizations can add their own.

Install a plugin by name:

```bash
claude plugin install my-plugin                  # from default marketplace
claude plugin install my-plugin@my-marketplace   # from a specific marketplace
claude plugin install my-plugin --scope project  # install at project scope
claude plugin install my-plugin --scope user     # install at user scope (default)
```

The `install` command resolves the plugin from the marketplace, copies it to the local plugin cache, and writes an entry to the appropriate `settings.json`.

### Enabling and disabling

Installed plugins are enabled by default. You can toggle a plugin without uninstalling it:

```bash
claude plugin enable my-plugin@my-marketplace
claude plugin disable my-plugin@my-marketplace
```

Or set it in `settings.json` directly:

```jsonc
{
  "enabledPlugins": {
    "my-plugin@my-marketplace": true,
    "other-plugin@my-marketplace": false
  }
}
```

### Uninstalling

```bash
claude plugin uninstall my-plugin@my-marketplace
```

This removes the plugin entry from the settings file and marks the cached version for cleanup. If other installed plugins declare a dependency on the plugin being removed, a warning is shown first.

### Built-in plugins

Claude Code ships with **built-in plugins** (identified by the `@builtin` suffix, e.g., `claude-in-chrome@builtin`). These appear in the `/plugin` UI alongside marketplace plugins but live inside the CLI binary — there is nothing to download. Built-in plugins are enabled by default and can be disabled in user settings.

### Reloading plugins

After installing or updating a plugin, run `/reload-plugins` in the REPL to apply the changes to the current session without restarting.

---

## Skills

### What a skill is

A skill is a slash command that, when invoked, injects a stored Markdown prompt into the conversation. Claude then responds to that prompt with access to the full tool set (subject to any `allowedTools` restriction declared by the skill).

Skills are a lightweight way to package repeatable workflows — code review patterns, deployment checklists, documentation generators — and share them with a team.

### Invoking a skill

```
/skillname
/skillname some arguments
```

Arguments are passed as a string to the skill's prompt template. The prompt can interpolate the argument string however the skill author chooses.

Skills appear in the `/` tab-completion menu in the REPL and are listed by `/help`.

### Skill sources

Skills are loaded from four sources, in order of increasing precedence:

1. **Bundled skills** — compiled into the CLI binary; always available to all users
2. **Built-in plugin skills** — skills contributed by built-in plugins (toggleable by the user)
3. **Marketplace plugin skills** — skills contributed by installed plugins
4. **Project skills** — Markdown files in `.claude/commands/` in the project directory (highest precedence; project-specific)

Duplicate names: a project skill with the same name as a bundled skill overrides the bundled one.

### Project skills

Create a file at `.claude/commands/<skillname>.md` to define a project-specific skill:

```
.claude/
  commands/
    deploy.md         # /deploy
    review.md         # /review
    update-tests.md   # /update-tests
```

The filename (without `.md`) becomes the slash command name. The Markdown content is the prompt. Subdirectories create namespaced commands: `commands/db/migrate.md` → `/db:migrate`.

### Bundled skill definition

Skills registered programmatically (by the CLI or built-in plugins) use the `BundledSkillDefinition` type:

```typescript
{
  name: string              // slash command name
  description: string       // shown in /help
  aliases?: string[]        // alternate invocation names
  whenToUse?: string        // hint to the model about when to auto-invoke
  argumentHint?: string     // shown in tab-completion
  allowedTools?: string[]   // restrict which tools the skill can use
  model?: string            // override the main loop model for this skill
  userInvocable?: boolean   // if false, hidden from /help (model-only)
  context?: 'inline' | 'fork'  // execution context
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}
```

### Skill hooks

Skills can declare their own hooks that run only while that skill is active:

```typescript
{
  name: 'my-skill',
  hooks: {
    PreToolUse: [{ matcher: 'Bash', hooks: [{ type: 'command', command: './pre-bash.sh' }] }]
  },
  getPromptForCommand: async (args) => [{ type: 'text', text: '...' }]
}
```

---

## Marketplace configuration

Add extra marketplaces in settings:

```jsonc
{
  "extraKnownMarketplaces": {
    "my-org": {
      "url": "https://github.com/my-org/claude-plugins",
      "branch": "main"
    }
  }
}
```

Marketplaces declared in `settings.json` are reconciled in the background at startup. If a new marketplace is detected, plugins from it are downloaded automatically. If an existing marketplace has updates, the session is marked as needing a reload (`/reload-plugins`).

Enterprise administrators can lock down which marketplaces are accessible via `policySettings`.
