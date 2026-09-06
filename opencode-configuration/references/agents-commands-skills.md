# OpenCode Agents, Commands & Skills Reference

Source of truth: `packages/core/src/v1/config/agent.ts`, `command.ts`, `skills.ts` and `packages/opencode/src/config/agent.ts`, `command.ts`.

## Agents

### Built-in agents

| Agent | Mode | Purpose |
| --- | --- | --- |
| `build` | primary | Default agent; all tools enabled (default_agent fallback) |
| `plan` | primary | Planning/analysis; edits and bash gated to `ask` by default |
| `general` | subagent | General-purpose; full tool access except todo; parallel work units |
| `explore` | subagent | Fast read-only codebase exploration |
| `scout` | subagent | Read-only external docs / dependency research |
| `compaction` | primary (hidden) | Compacts long context automatically |
| `title` | primary (hidden) | Generates session titles |
| `summary` | primary (hidden) | Generates session summaries |

Custom agents defined in config or markdown supplement these; same-name entries override built-ins. Disable any built-in with `"disable": true`.

### Agent fields

| Field | Type | Description |
| --- | --- | --- |
| `model` | string | `provider/model-id` override for this agent |
| `variant` | string | Default model variant (applies when using the agent's configured model) |
| `prompt` | string | System prompt; supports `{file:./path}` relative to the config file |
| `description` | string | When to use this agent (required for useful subagent routing; shown in `@` autocomplete) |
| `mode` | string | `primary` \| `subagent` \| `all` |
| `temperature` | number | 0.0–1.0; unset uses model default (typically 0; 0.55 for Qwen) |
| `top_p` | number | Nucleus sampling |
| `steps` | number | Max agentic iterations before forced text-only response (alias `maxSteps` deprecated) |
| `color` | string | Hex (`#FF5733`) or theme color (`primary`, `secondary`, `accent`, `success`, `warning`, `error`, `info`) |
| `disable` | bool | Disable the agent entirely |
| `hidden` | bool | Hide from `@` autocomplete (subagents only) |
| `options` | record | Provider-specific request options, e.g. `{ "reasoningEffort": "high" }` |
| `permission` | object | Permission rules merged over global; agent rules take precedence |
| `tools` | record | **Deprecated** — use `permission`. `true` ≈ `{"*": "allow"}`, `false` ≈ `{"*": "deny"}`; `write`/`edit`/`patch` collapse to `edit` |

Unknown keys in an agent object are collected into `options` automatically.

### Markdown agents

File name (without `.md`) becomes the agent name; nested directories create grouped names. Body becomes the `prompt`; frontmatter carries config.

Locations:
- Global: `~/.config/opencode/agents/*.md` (also `agent/`)
- Project: `.opencode/agents/*.md` (walks cwd up to git worktree)

```markdown
---
description: Reviews code for quality and best practices
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
permission:
  edit: deny
  bash: deny
---
You are in code review mode. Focus on:
- Code quality and best practices
- Potential bugs and edge cases
- Security considerations
```

The deprecated `{mode,modes}/*.md` layout still loads, forced to `mode: primary`.

### Using agents

- Primary agents: cycle with `tab` / `shift+tab`, or `<leader>a` for the agent list
- Subagents: `@name` mention in a message, or invoked automatically based on `description`
- Subagent sessions: `<leader>down` first child, `right`/`left` cycle children, `up` to parent
- `subagent_depth` (top-level config) controls nesting (default 1)

## Commands

Slash commands send a canned prompt. Built-ins include `/init`, `/undo`, `/redo`, `/share`, `/help`; custom commands with the same name override built-ins.

### Command fields

| Field | Type | Description |
| --- | --- | --- |
| `template` | string | The prompt sent to the LLM (required; markdown body in file form) |
| `description` | string | Shown in the TUI command list |
| `agent` | string | Agent to execute the command (subagent → subagent invocation by default) |
| `model` | string | Model override |
| `variant` | string | Variant override |
| `subtask` | bool | Force subagent invocation even for primary agents; set `false` to keep subagent-named commands in-context |

### Locations

- Global: `~/.config/opencode/commands/*.md` (also `command/`)
- Project: `.opencode/commands/*.md`
- JSON: `command` record in any config layer

### Template features

| Syntax | Expands to |
| --- | --- |
| `$ARGUMENTS` | All arguments passed after the command |
| `$1` … `$n` | Positional arguments |
| `` !`command` `` | Shell command output, run in the project root |
| `@path/to/file` | File contents injected into the prompt |

```markdown
---
description: Create a new component
---
Create a React component named $ARGUMENTS with TypeScript support.
Recent commits: !`git log --oneline -5`
Follow the patterns in @src/components/Button.tsx
```

## Skills

Reusable instructions discovered from `SKILL.md` files, loaded on demand via the native `skill` tool (agents see name + description, then load full content when relevant).

### Discovery locations

- `.opencode/skills/<name>/SKILL.md` (project; walks cwd to worktree)
- `~/.config/opencode/skills/<name>/SKILL.md` (global)
- Claude-compatible: `.claude/skills/`, `~/.claude/skills/`
- Agent-compatible: `.agents/skills/`, `~/.agents/skills/`
- Extra: `skills.paths` (additional folders) and `skills.urls` (remote skill endpoints, e.g. `https://example.com/.well-known/skills/`)

### Frontmatter

| Field | Required | Rules |
| --- | --- | --- |
| `name` | yes | 1–64 chars, `^[a-z0-9]+(-[a-z0-9]+)*$`, must match the containing directory name |
| `description` | yes | 1–1024 chars; specific enough for correct agent selection |
| `license` | no | e.g. MIT |
| `compatibility` | no | e.g. opencode |
| `metadata` | no | string-to-string map |

Unknown frontmatter fields are ignored.

### Gating

- `permission.skill` pattern rules: `allow` loads immediately, `deny` hides entirely, `ask` prompts (wildcards supported: `internal-*`)
- Per-agent overrides via agent `permission.skill` or (legacy) `tools: { skill: false }`
- Debug discovery: `opencode debug skill`
