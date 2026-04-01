# Tools Reference

All built-in tools registered in `src/tools.ts` via `getAllBaseTools()`. Feature-gated tools only appear when the corresponding flag is active.

| Name | Aliases | Category | Input Schema Summary | Description |
|------|---------|----------|---------------------|-------------|
| `Agent` | `Task` | Core | `prompt`, `description`, `subagent_type?`, `model?`, `run_in_background?`, `isolation?`, `cwd?` | Launch a new agent (subagent delegation) |
| `TaskOutput` | `AgentOutputTool`, `BashOutputTool` | Core | `task_id` | Retrieve output from a background task |
| `Bash` | — | Core | `command`, `description?`, `timeout?` | Execute shell commands |
| `Glob` | — | Search | `pattern`, `path?` | Fast file pattern matching via glob syntax |
| `Grep` | — | Search | `pattern`, `path?`, `glob?`, `type?`, `output_mode?`, `-A?`, `-B?`, `-C?`, `-i?` | Regex content search built on ripgrep |
| `ExitPlanMode` | — | Planning | *(no required fields)* | Exit plan mode and switch to execution |
| `Read` | — | Files | `file_path`, `offset?`, `limit?`, `pages?` | Read a file from the local filesystem |
| `Edit` | — | Files | `file_path`, `old_string`, `new_string`, `replace_all?` | Perform exact string replacements in files |
| `Write` | — | Files | `file_path`, `content` | Write a file to the local filesystem |
| `NotebookEdit` | — | Files | `notebook_path`, `cell_id`, `new_source?`, `cell_type?`, `edit_mode` | Edit Jupyter notebook cells |
| `WebFetch` | — | Web | `url`, `prompt` | Fetch and process web content |
| `WebSearch` | — | Web | `query`, `allowed_domains?`, `blocked_domains?` | Search the web via Anthropic's search API |
| `TodoWrite` | — | Tasks | `todos` | Write/update the session todo checklist |
| `TaskStop` | `KillShell` | Tasks | `task_id?`, `shell_id?` | Stop a background task or shell |
| `AskUserQuestion` | — | Interaction | `questions` (1–4 items with text, optional choices) | Prompt the user with structured questions |
| `Skill` | — | Skills | `command_name`, `_PROTO_skill_name?` | Invoke a named slash-command skill |
| `EnterPlanMode` | — | Planning | *(no required fields)* | Switch the session into plan mode |
| `SendMessage` | — | Multi-agent | `to`, `message`, `wait_for_reply?` | Send a message to a named teammate agent |
| `Brief` (alias) | `SendUserMessage` | Interaction | `message`, `heading?` | Send a visible message to the user |
| `BriefTool` | `Brief` | Interaction | `message`, `heading?` | Send a message to the user (primary name: `SendUserMessage`) |
| `Config` | — | Internal | `key`, `value?`, `action` | Read/write Claude Code config (ant-only) |
| `LSP` | — | IDE | `action`, `file_path?`, `position?` | Interact with Language Server Protocol servers |
| `EnterWorktree` | — | Git | `name` | Switch active git worktree |
| `ExitWorktree` | — | Git | *(no required fields)* | Return from a worktree to the main tree |
| `ListMcpResourcesTool` | — | MCP | `server_name?` | List resources available from MCP servers |
| `ReadMcpResourceTool` | — | MCP | `server_name`, `uri` | Read a specific MCP server resource |
| `ToolSearch` | — | Meta | `query`, `max_results?` | Search available tools by keyword |
| `TaskCreate` | — | Tasks v2 | `subject`, `description` | Create a new task (TodoV2 enabled only) |
| `TaskGet` | — | Tasks v2 | `task_id` | Retrieve details of a task |
| `TaskUpdate` | — | Tasks v2 | `task_id`, `status?`, `description?`, `metadata?` | Update a task's status or description |
| `TaskList` | — | Tasks v2 | *(no fields)* | List all tasks in the current task list |
| `TeamCreate` | — | Multi-agent | `team_name`, `description?`, `agent_type?` | Create a new agent team (swarms enabled) |
| `TeamDelete` | — | Multi-agent | *(no fields)* | Disband the current agent team |
| `Sleep` | — | Automation | `duration_ms` | Sleep for given milliseconds (PROACTIVE/KAIROS) |
| `CronCreate` | — | Automation | `schedule`, `prompt`, `description?` | Create a cron-scheduled trigger (AGENT_TRIGGERS) |
| `CronDelete` | — | Automation | `cron_id` | Delete a scheduled cron trigger |
| `CronList` | — | Automation | *(no fields)* | List active cron triggers |
| `RemoteTrigger` | — | Automation | `trigger_id`, `payload?` | Fire a remote trigger endpoint (AGENT_TRIGGERS_REMOTE) |
| `PowerShell` | — | Core | `command`, `description?` | Execute PowerShell commands (Windows only) |
| `WebBrowser` | — | Web | *(varies)* | Browser automation (WEB_BROWSER_TOOL flag) |
| `MonitorTool` | — | Automation | *(varies)* | Background monitoring tool (MONITOR_TOOL flag) |
| `SendUserFile` | — | Interaction | `file_path` | Send a file to the user (KAIROS flag) |
| `PushNotification` | — | Interaction | `message`, `title?` | Send a push notification (KAIROS flag) |
| `SubscribePR` | — | Automation | `pr_url` | Subscribe to GitHub PR webhooks (KAIROS_GITHUB_WEBHOOKS) |
| `SnipTool` | — | Context | *(varies)* | Snip/remove portions of conversation history (HISTORY_SNIP) |
| `ListPeers` | — | Multi-agent | *(no fields)* | List peer agents in UDS inbox (UDS_INBOX flag) |
| `WorkflowTool` | — | Automation | `workflow_name`, *(workflow-specific fields)* | Execute a bundled workflow script (WORKFLOW_SCRIPTS) |
| `REPL` | — | Core | `code`, `language?` | Execute code in a sandboxed REPL (ant-only) |
| `StructuredOutput` | — | Internal | *(synthetic)* | Synthetic output tool for structured responses |
| `TestingPermission` | — | Testing | *(varies)* | Permission test tool (NODE_ENV=test only) |
