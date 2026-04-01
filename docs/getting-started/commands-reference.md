# Commands Reference

> **Note:** Some commands listed here are feature-gated and may not be available in all builds. Feature-gated commands are only active when the corresponding flag is enabled (e.g. `VOICE_MODE`, `KAIROS`, `BRIDGE_MODE`, `KAIROS_BRIEF`, `CCR_REMOTE_SETUP`, `ULTRAPLAN`, `FORK_SUBAGENT`, `UDS_INBOX`, `BUDDY`, `KAIROS_GITHUB_WEBHOOKS`, `PROACTIVE`, `WORKFLOW_SCRIPTS`, `TORCH`, `HISTORY_SNIP`). Commands marked with an availability requirement (`claude-ai` or `console`) are only shown to users of the corresponding plan or API tier.

---

## Session & Navigation

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/clear` | `/reset`, `/new` | Clear conversation history and free up context | - |
| `/compact` | - | Clear conversation history but keep a summary in context. Optional: `/compact [instructions for summarization]` | - |
| `/resume` | `/continue` | Resume a previous conversation | - |
| `/rename` | - | Rename the current conversation | - |
| `/rewind` | `/checkpoint` | Restore the code and/or conversation to a previous point | - |
| `/branch` | `/fork` (when `FORK_SUBAGENT` off) | Create a branch of the current conversation at this point | - |
| `/exit` | `/quit` | Exit the REPL | - |
| `/export` | - | Export the current conversation to a file or clipboard | - |

## Information & Help

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/help` | - | Show help and available commands | - |
| `/status` | - | Show Claude Code status including version, model, account, API connectivity, and tool statuses | - |
| `/cost` | - | Show the total cost and duration of the current session | - |
| `/context` | - | Visualize current context usage as a colored grid | - |
| `/files` | - | List all files currently in context | - |
| `/stats` | - | Show your Claude Code usage statistics and activity | - |
| `/diff` | - | View uncommitted changes and per-turn diffs | - |
| `/insights` | - | Generate a report analyzing your Claude Code sessions | - |
| `/release-notes` | - | View release notes | - |

## Configuration & Settings

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/config` | `/settings` | Open config panel | - |
| `/model` | - | Set the AI model for Claude Code | - |
| `/effort` | - | Set effort level for model usage | - |
| `/theme` | - | Change the theme | - |
| `/color` | - | Set the prompt bar color for this session | - |
| `/vim` | - | Toggle between Vim and Normal editing modes | - |
| `/keybindings` | - | Open or create your keybindings configuration file | - |
| `/output-style` | - | Deprecated: use `/config` to change output style | - |
| `/statusline` | - | Set up Claude Code's status line UI | - |
| `/terminal-setup` | - | Install Shift+Enter key binding for newlines (terminal-dependent) | - |
| `/advisor` | - | Configure the advisor model | - |

## Memory & Context

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/memory` | - | Edit Claude memory files | - |
| `/init` | - | Initialize a new CLAUDE.md file with codebase documentation | - |
| `/add-dir` | - | Add a new working directory | - |
| `/tag` | - | Toggle a searchable tag on the current session | - |

## Tools & Permissions

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/permissions` | `/allowed-tools` | Manage allow & deny tool permission rules | - |
| `/hooks` | - | View hook configurations for tool events | - |
| `/mcp` | - | Manage MCP servers | - |
| `/sandbox` | - | Toggle sandbox execution mode | - |
| `/plan` | - | Enable plan mode or view the current session plan | - |

## Agents & Tasks

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/agents` | - | Manage agent configurations | - |
| `/tasks` | `/bashes` | List and manage background tasks | - |
| `/skills` | - | List available skills | - |
| `/fork` | - | Fork a subagent from the current conversation | `FORK_SUBAGENT` |
| `/peers` | - | Manage peer connections via Unix Domain Socket inbox | `UDS_INBOX` |
| `/buddy` | - | Buddy agent feature | `BUDDY` |

## AI Reviews & Analysis

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/review` | - | Review a pull request | - |
| `/ultrareview` | - | ~10-20 min deep review that finds and verifies bugs in your branch. Runs in Claude Code on the web | - |
| `/security-review` | - | Complete a security review of the pending changes on the current branch | - |
| `/ultraplan` | - | ~10-30 min · Claude Code on the web drafts an advanced plan you can edit and approve | `ULTRAPLAN` |

## Remote & Integrations

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/session` | `/remote` | Show remote session URL and QR code | - |
| `/ide` | - | Manage IDE integrations and show status | - |
| `/desktop` | `/app` | Continue the current session in Claude Desktop | - |
| `/mobile` | `/ios`, `/android` | Show QR code to download the Claude mobile app | - |
| `/install-github-app` | - | Set up Claude GitHub Actions for a repository | - |
| `/install-slack-app` | - | Install the Claude Slack app | - |
| `/pr-comments` | - | Get comments from a GitHub pull request | - |
| `/remote-env` | - | Configure the default remote environment for teleport sessions | - |
| `/remote-control` | `/rc` | Connect this terminal for remote-control sessions | `BRIDGE_MODE` |
| `/web-setup` | - | Setup Claude Code on the web (requires connecting your GitHub account) | `CCR_REMOTE_SETUP` (`claude-ai` only) |
| `/chrome` | - | Claude in Chrome (Beta) settings | - |

## Account & Billing

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/login` | - | Sign in with your Anthropic account (or switch accounts) | Not available when using 3rd-party services |
| `/logout` | - | Sign out from your Anthropic account | Not available when using 3rd-party services |
| `/usage` | - | Show plan usage limits | - |
| `/extra-usage` | - | Configure extra usage to keep working when limits are hit | - |
| `/rate-limit-options` | - | Show options when rate limit is reached | - |
| `/upgrade` | - | Upgrade to Max for higher rate limits and more Opus | - |
| `/passes` | - | Share a free week of Claude Code with friends | - |
| `/privacy-settings` | - | View and update your privacy settings | - |

## Plugins & Extensions

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/plugin` | `/plugins`, `/marketplace` | Manage Claude Code plugins | - |
| `/reload-plugins` | - | Activate pending plugin changes in the current session | - |

## Workflows

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/workflows` | - | Manage and run workflow scripts | `WORKFLOW_SCRIPTS` |

## AI Modes (Experimental/Gated)

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/voice` | - | Toggle voice mode | `VOICE_MODE` |
| `/fast` | - | Toggle fast mode (lightweight model only) | `VOICE_MODE`-era; `claude-ai` or `console` availability |
| `/brief` | - | Toggle brief-only mode | `KAIROS` or `KAIROS_BRIEF` |
| `/assistant` | - | Kairos assistant mode | `KAIROS` |
| `/proactive` | - | Proactive suggestions mode | `PROACTIVE` or `KAIROS` |
| `/torch` | - | Torch mode | `TORCH` |

## Utility

| Command | Aliases | Description | Feature Flag |
|---------|---------|-------------|--------------|
| `/copy` | - | Copy Claude's last response to clipboard (or `/copy N` for the Nth-latest) | - |
| `/btw` | - | Ask a quick side question without interrupting the main conversation | - |
| `/feedback` | `/bug` | Submit feedback about Claude Code | - |
| `/stickers` | - | Order Claude Code stickers | - |
| `/think-back` | - | Your 2025 Claude Code Year in Review | - |
| `/thinkback-play` | - | Play the thinkback animation | - |
| `/doctor` | - | Diagnose and verify your Claude Code installation and settings | - |
| `/upgrade` | - | Upgrade to Max for higher rate limits and more Opus | - |
| `/subscribe-pr` | - | Subscribe to pull request events via GitHub webhooks | `KAIROS_GITHUB_WEBHOOKS` |

---

## Internal-Only Commands

The following commands exist in the codebase but are **excluded from public builds**. They are only available in internal Anthropic builds (`USER_TYPE=ant`):

`/backfill-sessions`, `/break-cache`, `/bughunter`, `/commit`, `/commit-push-pr`, `/ctx-viz`, `/good-claude`, `/issue`, `/init-verifiers`, `/force-snip` (`HISTORY_SNIP`), `/mock-limits`, `/bridge-kick`, `/version`, `/ultraplan` (in internal list; also feature-gated), `/subscribe-pr` (in internal list; also feature-gated), `/reset-limits`, `/onboarding`, `/share`, `/summary`, `/teleport`, `/ant-trace`, `/perf-issue`, `/env`, `/oauth-refresh`, `/debug-tool-call`, `/agents-platform`, `/autofix-pr`
