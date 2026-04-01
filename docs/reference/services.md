# Services Reference

All service directories in `src/services/` plus significant top-level service files. Services encapsulate external I/O, background processing, and cross-cutting infrastructure.

| Service | Directory | Purpose | External Systems |
|---------|-----------|---------|-----------------|
| `AgentSummary` | `services/AgentSummary/` | Periodic background summarization of sub-agent conversations in coordinator mode; generates 1–2 sentence progress summaries | Anthropic API (forked sub-agent) |
| `analytics` | `services/analytics/` | Event logging pipeline. Queues events until a sink is attached; routes to Datadog and 1P event logger. Includes GrowthBook feature-gate client | Datadog, Anthropic 1P event logging, GrowthBook / Statsig |
| `api` | `services/api/` | Core Anthropic API client: streaming message requests, retry logic, admin requests, files API, session ingress, usage tracking, VCR playback | Anthropic Messages API (`@anthropic-ai/sdk`) |
| `autoDream` | `services/autoDream/` | Background memory consolidation; fires `/dream` prompt as a forked sub-agent when time and session-count gates pass | Anthropic API (forked agent) |
| `compact` | `services/compact/` | Context compaction: automatic and manual summarization, micro-compaction, cache-aware compaction, warning banners | Anthropic API (summary generation) |
| `extractMemories` | `services/extractMemories/` | Extracts durable memories from session transcripts at end of each query loop and writes them to `~/.claude/projects/…/memory/` | Local filesystem; Anthropic API (forked agent) |
| `lsp` | `services/lsp/` | Language Server Protocol client manager: spawns LSP server processes, tracks diagnostics, manages server lifecycle | LSP server processes (stdio); `vscode-jsonrpc` |
| `MagicDocs` | `services/MagicDocs/` | Automatically updates markdown files tagged with `# MAGIC DOC:` headers by running a forked sub-agent on file-read events | Anthropic API (forked agent); local filesystem |
| `mcp` | `services/mcp/` | Model Context Protocol client: connects to MCP servers (stdio, SSE, HTTP streaming), handles tools, resources, prompts, auth, and elicitation | MCP servers (`@modelcontextprotocol/sdk`); stdio, SSE, WebSocket |
| `oauth` | `services/oauth/` | OAuth 2.0 PKCE authorization code flow for Claude.ai login; manages tokens, refresh, and profile fetch | Anthropic OAuth endpoints; local browser |
| `plugins` | `services/plugins/` | Plugin installation, uninstallation, and CLI command surfacing | Local filesystem; plugin registry |
| `policyLimits` | `services/policyLimits/` | Fetches org-level feature restrictions from Anthropic API; disables CLI features for enterprise policy compliance | Anthropic API; local cache file |
| `PromptSuggestion` | `services/PromptSuggestion/` | Generates next-turn prompt suggestions using a forked sub-agent after each response | Anthropic API (forked agent) |
| `remoteManagedSettings` | `services/remoteManagedSettings/` | Fetches and caches enterprise-managed settings from Anthropic API (ETag-based, fail-open) | Anthropic API; local cache file |
| `SessionMemory` | `services/SessionMemory/` | Maintains a rolling markdown notes file about the current conversation via periodic forked sub-agent runs | Anthropic API (forked agent); local filesystem |
| `settingsSync` | `services/settingsSync/` | Syncs user settings and memory files between local and remote Claude Code environments (upload on interactive CLI, download on CCR) | Anthropic API; local filesystem |
| `teamMemorySync` | `services/teamMemorySync/` | Syncs shared team memory files per-repo between local filesystem and Anthropic server API; delta-upload semantics | Anthropic API; local filesystem; git remote metadata |
| `tips` | `services/tips/` | Tip registry, history tracking, and scheduled display of usage tips | Local tip history file |
| `tools` | `services/tools/` | Tool execution pipeline: `StreamingToolExecutor`, `toolExecution`, `toolHooks`, `toolOrchestration` | Tool implementations; analytics |
| `toolUseSummary` | `services/toolUseSummary/` | Generates human-readable summaries of tool use for compact/summary views | Analytics |
| `awaySummary` | `services/awaySummary.ts` | Generates a brief summary of what happened while the terminal was away | Anthropic API (forked agent) |
| `claudeAiLimits` | `services/claudeAiLimits.ts` | Fetches, caches, and enforces Claude.ai rate limits and usage caps | Anthropic API |
| `diagnosticTracking` | `services/diagnosticTracking.ts` | Tracks and surfaces LSP diagnostic events for analytics and UI | LSP diagnostic registry |
| `internalLogging` | `services/internalLogging.ts` | Internal structured log sink for debugging | Local log buffer |
| `mcpServerApproval` | `services/mcpServerApproval.tsx` | UI and logic for approving new MCP server connections | User interaction; settings |
| `mockRateLimits` | `services/mockRateLimits.ts` | Simulates rate limit responses for development/testing | — |
| `notifier` | `services/notifier.ts` | Sends system desktop notifications | macOS/Linux notification APIs |
| `preventSleep` | `services/preventSleep.ts` | Prevents system sleep while Claude Code is running | macOS `caffeinate` / Linux `systemd-inhibit` |
| `rateLimitMessages` | `services/rateLimitMessages.ts` | Formats and routes rate limit error messages to the UI | — |
| `tokenEstimation` | `services/tokenEstimation.ts` | Estimates token counts for messages without a full API call | Anthropic tokenizer |
| `vcr` | `services/vcr.ts` | VCR (record/replay) for API requests; used in testing | Anthropic API; local cassette files |
| `voice` | `services/voice.ts` | Voice recording interface (macOS native / SoX) | Native audio API; SoX |
| `voiceKeyterms` | `services/voiceKeyterms.ts` | Fetches domain-specific keyterms to improve voice STT accuracy | Anthropic API |
| `voiceStreamSTT` | `services/voiceStreamSTT.ts` | Streams audio to Anthropic's `voice_stream` endpoint for real-time STT | Anthropic voice_stream API |
