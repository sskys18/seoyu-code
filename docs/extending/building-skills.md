# Building Skills

Skills are Markdown files (or programmatic definitions) that Claude Code surfaces to the model as slash commands. When a user types `/my-skill` or the model is told to invoke `my-skill` via the Skill tool, Claude Code loads the skill's Markdown content and passes it to the model as the turn prompt.

## Skill File Format

A skill file is a Markdown document with an optional YAML frontmatter block:

```markdown
---
description: A brief description shown in skill listings
argument-hint: <query>
when_to_use: Use this skill when the user asks to search documentation
allowed-tools: Read, Grep, Bash(find *)
model: claude-opus-4-6
context: fork
---

# My Skill

Your task is to $ARGUMENTS.

Search the docs directory for relevant files and summarize them.
```

The filename (without `.md`) becomes the skill name and the slash-command trigger. A file at `.claude/skills/search-docs.md` registers as `/search-docs`.

## Frontmatter Fields

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `description` | string | Extracted from first heading | Short description shown in `/skills` listing and Skill tool schema |
| `argument-hint` | string | — | Placeholder shown in the UI (e.g. `<query>`) |
| `arguments` | string or list | — | Named argument definitions for `$ARG_NAME` substitution |
| `when_to_use` | string | — | Guidance to help the model decide when to trigger this skill automatically |
| `allowed-tools` | string | — | Comma-separated tool allowlist. Glob patterns allowed: `Bash(git *)` |
| `model` | string | inherit | Override the model for this skill's turn. Use `inherit` to use the session model |
| `effort` | string or int | — | Effort level: `low`, `medium`, `high`, or a token integer |
| `context` | `fork` | — | Run the skill in a forked subagent instead of inline |
| `agent` | string | — | Agent name when `context: fork` is set |
| `user-invocable` | bool | `true` | If `false`, the skill is hidden from user-facing listings |
| `disable-model-invocation` | bool | `false` | Run the skill as a pure prompt transform without calling the model |
| `hooks` | object | — | Inline hook definitions scoped to this skill's execution |
| `paths` | string or list | — | File-path glob patterns that restrict when the skill is active |
| `version` | string | — | Semver version for the skill |
| `name` | string | filename | Override the display name without changing the command trigger |
| `shell` | object | — | Shell execution configuration for pre/post command execution |

## Argument Substitution

Inside the Markdown body, `$ARGUMENTS` is replaced with the raw argument string passed by the user. Named arguments use `$ARG_NAME` syntax and are declared in `arguments` frontmatter:

```markdown
---
arguments: [filename, mode]
---

Open $filename in $mode mode.
```

Unused named arguments are substituted with empty string. If none of the named patterns match, the full argument string is injected at `$ARGUMENTS`.

## Skill Load Paths

Skills are loaded from multiple directories in priority order:

| Source | Directory |
|--------|-----------|
| Managed (policy) | `{managed-path}/.claude/skills/` |
| User | `~/.claude/skills/` |
| Project | `.claude/skills/` |
| Local | `.claude/skills/` (local overrides) |
| Plugin | Plugin's `skills/` directory |
| Bundled | Compiled into the CLI binary |
| MCP | Dynamically registered via MCP prompts |

Duplicate skill names are deduplicated by canonical file path. The first loaded wins.

## Bundled Skills (Programmatic)

Skills compiled into the CLI binary are registered with `registerBundledSkill` from `src/skills/bundledSkills.ts`:

```typescript
import { registerBundledSkill } from '../skills/bundledSkills.js'

registerBundledSkill({
  name: 'my-skill',
  description: 'Does something useful',
  whenToUse: 'Use when the user asks to do something useful',
  allowedTools: ['Read', 'Bash(echo *)'],
  argumentHint: '<topic>',
  // Optional: reference files extracted to a temp dir on first invocation
  files: {
    'reference/guide.md': '# Guide\n...',
  },
  async getPromptForCommand(args, context) {
    return [{ type: 'text', text: `Do something useful with: ${args}` }]
  },
})
```

`BundledSkillDefinition` fields map directly to the frontmatter keys above, with `getPromptForCommand` replacing the Markdown body. The `files` map is extracted to a secure per-process temp directory on first invocation; the model receives a `Base directory for this skill: <dir>` prefix in the prompt so it can `Read` those files.

## The Skill Tool

The model invokes skills through the `Skill` tool (also called `SkillTool` internally). The tool's input schema lists every registered skill with its `name`, `description`, and `whenToUse`. The model picks the best-matching skill and supplies arguments.

Claude Code then:
1. Loads the skill's full Markdown content (deferred until invocation).
2. Substitutes arguments.
3. Runs the prompt inline (default) or in a forked subagent (`context: fork`).
4. Returns the result to the model as a tool result.

## Rigid vs. Flexible Skill Patterns

**Rigid skills** have a fixed, fully-specified prompt. The model has no latitude — it executes exactly the instructions in the Markdown. Use `allowed-tools` to restrict which tools the model may call during the skill's execution.

```markdown
---
allowed-tools: Bash(git log --oneline -20)
disable-model-invocation: false
---

Run `git log --oneline -20` and summarize the last 20 commits.
```

**Flexible skills** provide a framework prompt and let the model decide how to fulfill the request. They expose broad tool access and rely on `when_to_use` to guide selection:

```markdown
---
allowed-tools: Read, Grep, Bash
when_to_use: Use when the user wants to explore or understand unfamiliar code
---

You are exploring a codebase. The user's query is: $ARGUMENTS

Start by reading the relevant files, then provide a clear explanation.
```

## Hooks in Skills

Frontmatter hooks fire around the skill's execution, not around individual tool calls within it:

```markdown
---
hooks:
  PreSkillExecution:
    - hooks:
        - type: command
          command: echo "Starting skill"
  PostSkillExecution:
    - hooks:
        - type: command
          command: echo "Skill done"
---
```

## Best Practices

- Keep `description` under 120 characters — it's shown in compact listings.
- Provide `when_to_use` for any skill you want the model to discover autonomously; without it the skill is only reachable by explicit slash-command.
- Use `argument-hint` to show users what input the skill expects.
- Prefer `context: fork` for skills that may generate many tool calls, to keep the parent context clean.
- Use `paths` to restrict domain-specific skills (e.g. `paths: ["**/*.py"]` for a Python-only skill).
- Avoid `disable-model-invocation: true` unless you need a pure prompt passthrough — the model cannot observe or respond to the prompt in this mode.
