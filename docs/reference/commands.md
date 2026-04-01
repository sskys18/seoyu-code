# Commands Reference

All public slash commands registered in `src/commands.ts` via `COMMANDS()`. Commands in `INTERNAL_ONLY_COMMANDS` are excluded unless `USER_TYPE=ant`. Feature-gated commands appear only when their flag is active.

| Name | Aliases | Args | Feature Flag / Availability | Description |
|------|---------|------|-----------------------------|-------------|
| `add-dir` | — | `<path>` | — | Add a new working directory |
| `advisor` | — | `[<model>\|off]` | ant-only (canUserConfigureAdvisor) | Configure the advisor model |
| `agents` | — | — | — | Manage agent configurations |
| `branch` | `fork` (when FORK_SUBAGENT off) | `[name]` | — | Create a branch of the current conversation |
| `btw` | — | `<question>` | — | Ask a quick side question without interrupting the main conversation |
| `chrome` | — | — | claude-ai only | Claude in Chrome (Beta) settings |
| `clear` | `reset`, `new` | — | — | Clear conversation history and free up context |
| `color` | — | `<color\|default>` | — | Set the prompt bar color for this session |
| `compact` | — | `[summarization instructions]` | — | Clear history but keep a summary in context |
| `config` | `settings` | — | — | Open config panel |
| `context` | — | — | — | Visualize current context usage as a colored grid |
| `copy` | — | `[N]` | — | Copy Claude's last response to clipboard |
| `cost` | — | — | Hidden for claude.ai subscribers | Show total cost and duration of the current session |
| `desktop` | `app` | — | claude-ai; macOS/Win64 | Continue the current session in Claude Desktop |
| `diff` | — | — | — | View uncommitted changes and per-turn diffs |
| `doctor` | — | — | — | Diagnose and verify Claude Code installation |
| `effort` | — | `[low\|medium\|high\|max\|auto]` | — | Set effort level for model usage |
| `exit` | `quit` | — | — | Exit the REPL |
| `export` | — | `[filename]` | — | Export the current conversation to a file or clipboard |
| `extra-usage` | — | — | claude-ai (extra usage allowed) | Configure extra usage to keep working when limits are hit |
| `fast` | — | `[on\|off]` | claude-ai or console; isFastModeEnabled | Toggle fast mode |
| `feedback` | `bug` | `[report]` | — | Submit feedback about Claude Code |
| `files` | — | — | ant-only | List all files currently in context |
| `heapdump` | — | — | — | Dump the JS heap to ~/Desktop |
| `help` | — | — | — | Show help and available commands |
| `hooks` | — | — | — | View hook configurations for tool events |
| `ide` | — | `[open]` | — | Manage IDE integrations and show status |
| `init` | — | — | — | Initialize a new project (creates CLAUDE.md) |
| `install-github-app` | — | — | claude-ai or console | Set up Claude GitHub Actions for a repository |
| `install-slack-app` | — | — | claude-ai | Install the Claude Slack app |
| `keybindings` | — | — | keybinding customization enabled | Open or create keybindings configuration file |
| `mcp` | — | `[enable\|disable [server-name]]` | — | Manage MCP servers |
| `memory` | — | — | — | Edit Claude memory files |
| `mobile` | `ios`, `android` | — | — | Show QR code to download the Claude mobile app |
| `model` | — | `[model]` | — | Set the AI model for Claude Code |
| `output-style` | — | — | — | Deprecated: use /config to change output style |
| `passes` | — | — | — | *(usage pass management)* |
| `permissions` | `allowed-tools` | — | — | Manage allow & deny tool permission rules |
| `plan` | — | `[open\|<description>]` | — | Enable plan mode or view current session plan |
| `plugin` | `plugins`, `marketplace` | — | — | Manage Claude Code plugins |
| `pr-comments` | — | `[PR number]` | — | Get comments from a GitHub pull request |
| `privacy-settings` | — | — | — | View and update privacy settings |
| `rate-limit-options` | — | — | — | Show options when rate limit is reached |
| `reload-plugins` | — | — | — | Reload all plugins |
| `remote-env` | — | — | — | Configure default remote environment for teleport |
| `rename` | — | `[name]` | — | Rename the current conversation |
| `resume` | `continue` | `[id or search term]` | — | Resume a previous conversation |
| `rewind` | `checkpoint` | — | — | Restore code and/or conversation to a previous point |
| `review` | — | `[PR number]` | — | Review a pull request (local AI review) |
| `sandbox` | — | `[exclude "command pattern"]` | — | Toggle sandbox mode |
| `security-review` | — | — | — | Complete a security review of pending branch changes |
| `session` | `remote` | — | Remote mode only | Show remote session URL and QR code |
| `skills` | — | — | — | List available skills |
| `stats` | — | — | — | Show Claude Code usage statistics and activity |
| `status` | — | — | — | Show Claude Code status (version, model, account, etc.) |
| `statusline` | — | — | — | Set up Claude Code's status line UI |
| `stickers` | — | — | — | Order Claude Code stickers |
| `tag` | — | `<tag-name>` | ant-only | Toggle a searchable tag on the current session |
| `tasks` | `bashes` | — | — | List and manage background tasks |
| `terminalSetup` | — | — | — | Configure terminal settings |
| `theme` | — | — | — | Change the theme |
| `think-back` | — | — | tengu_thinkback Statsig gate | Your 2025 Claude Code Year in Review |
| `think-back-play` | — | — | tengu_thinkback Statsig gate | Play back thinkback data |
| `ultrareview` | — | — | — | Deep bug-finding review (~10–20 min), runs in Claude Code on the web |
| `upgrade` | — | — | claude-ai | Upgrade to Max for higher rate limits |
| `usage` | — | — | claude-ai | Show plan usage limits |
| `insights` | — | — | — | Generate a report analyzing your Claude Code sessions |
| `vim` | — | — | — | Toggle between Vim and Normal editing modes |
| `login` | — | — | Non-3P services only | Log in to Claude |
| `logout` | — | — | Non-3P services only | Log out of Claude |
| `remote-setup` | — | — | CCR_REMOTE_SETUP flag | Set up Claude Code Remote environment |
| `fork` | — | — | FORK_SUBAGENT flag | Fork current session as a subagent |
| `buddy` | — | — | BUDDY flag | Buddy agent management |
| `proactive` | — | — | PROACTIVE or KAIROS flag | Proactive agent mode management |
| `brief` | — | — | KAIROS or KAIROS_BRIEF flag | Brief/notification management |
| `assistant` | — | — | KAIROS flag | Assistant management |
| `bridge` | — | — | BRIDGE_MODE flag | Bridge mode connection |
| `remote-control-server` | — | — | DAEMON + BRIDGE_MODE flags | Start the remote control server |
| `voice` | — | — | VOICE_MODE flag | Voice mode controls |
| `workflows` | — | — | WORKFLOW_SCRIPTS flag | Manage and run workflow scripts |
| `peers` | — | — | UDS_INBOX flag | Manage peer agent connections |
| `torch` | — | — | TORCH flag | Torch feature |
| `ultraplan` | — | — | ULTRAPLAN flag | Ultra planning mode (internal) |

**Internal-only commands** (visible only with `USER_TYPE=ant`): `backfill-sessions`, `break-cache`, `bughunter`, `commit`, `commit-push-pr`, `ctx_viz`, `good-claude`, `issue`, `init-verifiers`, `force-snip`, `mock-limits`, `bridge-kick`, `version`, `ultraplan`, `subscribe-pr`, `reset-limits`, `onboarding`, `share`, `summary`, `teleport`, `ant-trace`, `perf-issue`, `env`, `oauth-refresh`, `debug-tool-call`, `agents-platform`, `autofix-pr`.
