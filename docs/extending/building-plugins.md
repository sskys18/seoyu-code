# Building Plugins

Plugins are the primary extension mechanism for Claude Code. A plugin can bundle skills, hooks, MCP servers, and LSP servers into a single distributable unit that users can install from a marketplace or local path.

## Plugin ID Format

Every plugin has a globally unique identifier of the form `{name}@{marketplace}`. Built-in plugins (those that ship with the CLI) use the special sentinel marketplace `builtin`, producing IDs like `my-plugin@builtin`. Third-party marketplace plugins use the marketplace name registered in the user's configuration, such as `my-plugin@anthropic`.

## Plugin Directory Structure

A marketplace-installed plugin lives under `~/.claude/plugins/repos/{marketplace}/{name}/`. Its on-disk layout:

```
{plugin-name}/
  manifest.json          # Required — describes the plugin
  skills/                # Optional — Markdown skill files
    my-skill.md
  hooks/
    hooks.json           # Optional — hook definitions
  lsp/
    lsp.json             # Optional — LSP server definitions
  mcp/
    mcp.json             # Optional — MCP server definitions
```

## Manifest Format

`manifest.json` is required and must conform to `PluginManifest`:

```json
{
  "name": "my-plugin",
  "description": "A short human-readable description",
  "version": "1.0.0"
}
```

Fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Must match the directory name |
| `description` | string | yes | Shown in the `/plugin` UI |
| `version` | string | yes | Semver string |

## Registering Components

### Skills

Place Markdown files in the `skills/` directory. Each file is loaded by `loadSkillsDir` and presented to the model as a slash command. See [building-skills.md](./building-skills.md) for the full frontmatter reference.

### Hooks

Create `hooks/hooks.json`. The format mirrors the `hooks` key in `settings.json`:

```json
{
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        { "type": "command", "command": "echo pre-bash" }
      ]
    }
  ]
}
```

Hook definitions contributed by a plugin are tagged with source `pluginHook` and appear below user/project hooks in priority order.

### MCP Servers

Create `mcp/mcp.json` with an `mcpServers` map:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

All transport types supported by the core MCP client (`stdio`, `sse`, `http`, `ws`) are valid here. See [mcp-integration.md](./mcp-integration.md) for transport details.

### LSP Servers

Create `lsp/lsp.json`. LSP servers are exclusively plugin-provided — they cannot be configured in `settings.json` directly. See [lsp-integration.md](./lsp-integration.md) for the configuration schema.

## Registering Built-in Plugins Programmatically

Plugins that ship inside the CLI binary (rather than from a marketplace) are registered at startup via `registerBuiltinPlugin` from `src/plugins/builtinPlugins.ts`:

```typescript
import { registerBuiltinPlugin } from './plugins/builtinPlugins.js'

registerBuiltinPlugin({
  name: 'my-feature',
  description: 'Adds my feature to Claude Code',
  version: '1.0.0',
  defaultEnabled: true,
  isAvailable: () => process.env.ENABLE_MY_FEATURE === '1',
  skills: [mySkillDefinition],
  hooks: { PreToolUse: [...] },
  mcpServers: { 'my-server': { type: 'stdio', command: '...' } },
})
```

The `isAvailable()` predicate is evaluated at load time; returning `false` hides the plugin entirely from the UI and from hook/MCP registration.

`BuiltinPluginDefinition` fields:

| Field | Type | Notes |
|-------|------|-------|
| `name` | string | Unique within built-ins |
| `description` | string | Shown in `/plugin` UI |
| `version` | string | Semver |
| `defaultEnabled` | boolean | Default `true` |
| `isAvailable` | `() => boolean` | Optional; omit to always show |
| `skills` | `BundledSkillDefinition[]` | Inline skill definitions |
| `hooks` | `HooksSettings` | Hook config object |
| `mcpServers` | record | MCP server configs |

## Plugin Marketplace

A marketplace is a Git repository containing a registry of plugin entries. Claude Code clones and caches marketplaces under `~/.claude/plugins/marketplaces/`. The process:

1. **Declaration** — Users or projects add a marketplace URL to their settings.
2. **Background reconciliation** — On startup, `performBackgroundPluginInstallations` calls `reconcileMarketplaces` to diff declared vs. materialized marketplaces. Missing marketplaces are cloned; changed source URLs cause a re-clone.
3. **Plugin lookup** — When a user installs `name@marketplace`, `getPluginById` resolves the entry from the cached marketplace registry and downloads the versioned plugin archive.
4. **Versioned cache** — Installed archives are stored under `~/.claude/plugins/versions/{marketplace}/{name}/{version}/`. A symlink in `~/.claude/plugins/active/` points at the live version.

`pluginOperations.ts` exposes pure library functions for install, uninstall, enable, disable, and update — these are used by both the `/plugin` CLI commands and the interactive plugin management UI.

## Plugin Lifecycle

```
install → enable → reload → execute
```

1. **Install** — `installPlugin(identifier, scope)` resolves the plugin from its marketplace, downloads the archive, unpacks it to the versioned cache, and records the installation in the scope's `installed_plugins.json`.
2. **Enable** — Sets `enabledPlugins[pluginId] = true` in the target settings file. Plugins are enabled by default after install.
3. **Reload** — `/reload-plugins` (or the `refreshActivePlugins` function) clears all caches, re-reads every installed plugin from disk, re-establishes MCP connections, and bumps `pluginReconnectKey` in app state.
4. **Execute** — Skills are available as slash commands; hooks fire on matching events; MCP servers provide tool implementations.

## Installation Scopes

| Scope | Settings file | Visibility |
|-------|--------------|------------|
| `user` | `~/.claude/settings.json` | All projects |
| `project` | `.claude/settings.json` | This project |
| `local` | `.claude/settings.local.json` | Local override, not committed |
| `managed` | Policy-managed path | Read-only; admin-deployed |

## Policy Controls

Admins can restrict plugins with `allowManagedHooksOnly` in policy settings. When set, only managed hooks execute and the user-facing hook list is empty. Individual plugins can be blocked by name via plugin policy configuration.
