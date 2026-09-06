---
name: opencode-configuration
description: Configure and customize OpenCode (opencode-ai) coding agent settings, providers and models, agents, permissions, MCP servers, commands, skills, plugins, TUI themes, and keybinds. TRIGGERS - opencode config, configure opencode, opencode.json, opencode settings, opencode providers, custom models, whitelist, opencode agents, opencode permissions, opencode mcp, opencode keybinds, tui.json, opencode plugins, opencode skills, opencode commands.
---

# OpenCode Configuration

A comprehensive guide and reference playbook for configuring, customizing, and troubleshooting **OpenCode** (opencode-ai).

## When to Use This Skill

Use this skill whenever you need to:
- Configure OpenCode global settings (`~/.config/opencode/opencode.jsonc`) or project settings (`opencode.json` in a repo)
- Add providers, custom models, or local models — including the `whitelist` requirement
- Restrict visible providers via `enabled_providers` / `disabled_providers`
- Configure agents and subagents (JSON or markdown) — models, prompts, steps, colors
- Tune the permission system (`allow` / `ask` / `deny`, granular rules, `external_directory`)
- Add, update, or troubleshoot MCP servers (local stdio, remote HTTP, OAuth)
- Create slash commands (`/command`), agent skills (`SKILL.md`), or install plugins
- Customize the TUI: themes, keybinds, leader key, attention sounds (`tui.json`)
- Tune context compaction, tool output truncation, or image attachment limits
- Sync OpenCode configuration across machines using chezmoi

---

## Architecture & Configuration Hierarchy

| Scope | Location | Purpose & Precedence |
| --- | --- | --- |
| **macOS Managed Preferences** | `ai.opencode.managed` domain (`.mobileconfig` via MDM) | Highest priority; not user-overridable |
| **Managed Config Files** | `/Library/Application Support/opencode/` (macOS), `/etc/opencode/` (Linux), `%ProgramData%\opencode` (Windows) | Admin-controlled; requires root to write |
| **Inline Config** | `OPENCODE_CONFIG_CONTENT` env var | Runtime overrides as raw JSON string |
| **`.opencode` directories** | `<repo>/.opencode/` (walks cwd up to git worktree) + `~/.opencode/` | Agents, commands, skills, plugins, `opencode.json(c)`, `tui.json(c)` |
| **Project Config** | `opencode.json` / `opencode.jsonc` in project (walks up to git worktree) | Project-specific settings; overrides global |
| **Custom Config** | `OPENCODE_CONFIG` env var (file path) | Custom overrides between global and project |
| **Global Config** | `~/.config/opencode/opencode.jsonc` (or `opencode.json`, legacy `config.json`) | Persistent user-wide configuration |
| **Remote Config** | `.well-known/opencode` endpoint of authenticated providers | Organizational defaults; lowest priority |

Later sources override earlier ones **only for conflicting keys** — configs are deep-merged, not replaced.

**Merge rules:**
- Objects are deep-merged; non-conflicting keys from all layers survive.
- Arrays are **replaced wholesale** (never appended) — with one exception: `instructions` is concatenated and deduplicated across layers.
- Within the global scope, files load in order `config.json` → `opencode.json` → `opencode.jsonc` (so `opencode.jsonc` wins among globals).

### Key Config Files & Directories

| Path | Format | Purpose |
| --- | --- | --- |
| `~/.config/opencode/opencode.jsonc` | JSONC | Main settings: providers, agents, permissions, MCP, models |
| `~/.config/opencode/tui.json` | JSON | TUI settings: theme, keybinds, leader, attention, cursor |
| `~/.config/opencode/agents/*.md` | Markdown+frontmatter | Global custom agents |
| `~/.config/opencode/commands/*.md` | Markdown+frontmatter | Global custom commands |
| `~/.config/opencode/skills/*/SKILL.md` | Markdown+frontmatter | Global agent skills |
| `~/.config/opencode/plugins/*.{ts,js}` | JS/TS | Global local plugins (auto-loaded) |
| `~/.config/opencode/themes/*.json` | JSON | Custom TUI themes |
| `~/.config/opencode/package.json` | JSON | Dependencies for local plugins (installed via bun at startup) |
| `~/.local/share/opencode/auth.json` | JSON | Provider credentials (managed by `opencode auth login`) |
| `<repo>/opencode.json` or `<repo>/.opencode/` | JSONC / mixed | Project-scoped equivalents of all the above |

Use `$schema`: `https://opencode.ai/config.json` (main) and `https://opencode.ai/tui.json` (TUI) for editor validation and completion.

### Relevant Environment Variables

| Variable | Effect |
| --- | --- |
| `OPENCODE_CONFIG` | Load an extra config file (between global and project precedence) |
| `OPENCODE_CONFIG_DIR` | Extra directory searched like `.opencode/` (agents, commands, plugins) |
| `OPENCODE_CONFIG_CONTENT` | Inline config JSON string, merged as project-local scope |
| `OPENCODE_DISABLE_PROJECT_CONFIG` | Skip all project config discovery |
| `OPENCODE_TUI_CONFIG` | Custom TUI config file path |
| `OPENCODE_PERMISSION` | JSON permission overrides merged on top of everything |
| `OPENCODE_DISABLE_AUTOCOMPACT` / `OPENCODE_DISABLE_PRUNE` | Force-disable compaction/pruning |

---

## Quick Reference CLI & TUI Commands

```bash
opencode                                # Start the TUI in the current project
opencode run "message"                  # Headless one-shot run (--auto to auto-approve)
opencode serve                          # Start the local server (config: server.port/hostname)
opencode web                            # Start the web UI

# Providers & models
opencode auth login [provider]          # Authenticate a provider (alias of opencode providers login)
opencode auth logout [provider]         # Remove stored credentials
opencode providers list                 # List providers and credential status
opencode models [provider]              # List available models (--verbose, --refresh)

# MCP
opencode mcp list                       # List MCP servers and connection status
opencode mcp add [name]                 # Add an MCP server interactively
opencode mcp auth [name]                # Authenticate an OAuth-enabled remote MCP server
opencode mcp logout [name]              # Remove stored MCP OAuth credentials
opencode mcp debug <name>               # Debug OAuth connection for a server

# Plugins, agents, debugging
opencode plugin <module>                # Install a plugin and update config
opencode agent list / create            # List or create agents
opencode debug config                   # Print the fully resolved config (best diagnostic)
opencode debug paths                    # Print all discovered config paths
opencode debug skill                    # Debug skill discovery
opencode upgrade / uninstall            # Self-management
```

Key TUI slash commands: `/connect` (add provider credentials), `/models`, `/agents`, `/commands` (or `ctrl+p` palette), `/mcp`, `/plugins`, `/init`, `/undo`, `/redo`, `/share`, `/help`.

---

## 1. Main Settings (`opencode.jsonc`)

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // Models
  "model": "plexus/gpt-5.6-luna",              // provider/model-id
  "small_model": "plexus/gemini-3.5-flash-lite", // titles, summaries (falls back to model)
  "default_agent": "build",
  "subagent_depth": 2,                          // max subagent nesting (default 1)

  // Provider visibility (CRITICAL — see section 2)
  "enabled_providers": ["plexus"],              // when set, ONLY these load

  // Behavior
  "share": "disabled",                          // manual | auto | disabled
  "autoupdate": false,                          // true | false | "notify"
  "snapshot": true,                             // filesystem snapshots for /undo /redo
  "username": "matt",                           // display name override
  "shell": "/bin/zsh",                          // shell for terminal + bash tool
  "logLevel": "INFO",                           // DEBUG | INFO | WARN | ERROR
  "instructions": ["~/.agents/APPEND_SYSTEM.md"], // extra instruction files (concatenates across layers)

  // Tool toggles (legacy; prefer "permission")
  "tools": { "websearch": false, "webfetch": false },

  // Context management
  "compaction": { "auto": true, "prune": false, "tail_turns": 5 },
  "tool_output": { "max_lines": 1000, "max_bytes": 65535 },

  // Attachments
  "attachment": { "image": { "max_width": 1024, "max_height": 768 } },

  // Experimental
  "experimental": { "batch_tool": true, "continue_loop_on_deny": true }
}
```

Full settings catalog: [references/settings.md](references/settings.md)

---

## 2. Providers, Custom Models & Whitelists

> **CRITICAL RULE:** When adding a new provider or custom model, the provider config MUST include a `whitelist` array listing the model IDs, and the provider must be listed in top-level `enabled_providers`. Without both, the models will **not** appear in the `/models` selector.

```jsonc
{
  "enabled_providers": ["plexus"],
  "provider": {
    "plexus": {
      "name": "Plexus Gateway",
      "options": {
        "plexusBaseURL": "https://plexus.home.cowger.us",
        "apiKey": "{env:AIHOME_API_KEY}:opencode"
      },
      "whitelist": ["gpt-5.6-luna", "gpt-5.6-terra", "gemini-3.5-flash-lite"]
    },
    "local-vllm": {
      "npm": "@ai-sdk/openai-compatible",       // AI SDK package for OpenAI-compatible APIs
      "name": "vLLM (local)",
      "options": { "baseURL": "http://127.0.0.1:8000/v1", "apiKey": "none" },
      "whitelist": ["Qwen/Qwen2.5-Coder-32B-Instruct"],
      "models": {
        "Qwen/Qwen2.5-Coder-32B-Instruct": {
          "name": "Qwen 2.5 Coder 32B",
          "limit": { "context": 65536, "output": 8192 },
          "tool_call": true,
          "attachment": false
        }
      }
    }
  }
}
```

- `whitelist` — hides every model except those listed (IDs as shown in `/models`)
- `blacklist` — removes specific listed models; combinable (whitelist narrows, then blacklist removes)
- `options.baseURL` / `options.apiKey` — endpoint override and credential
- `options.timeout` (ms, or `false` to disable), `options.chunkTimeout` (ms between SSE chunks), `options.setCacheKey` (prompt caching)
- Override an existing provider's model metadata via `models`: `limit.context/output`, `cost.input/output/cache_read/cache_write`, `reasoning`, `attachment`, `modalities`, `status`, `variants`

Credentials added via `/connect` or `opencode auth login` are stored in `~/.local/share/opencode/auth.json` — never commit that file.

Deep dive: [references/providers-models.md](references/providers-models.md)

---

## 3. Agents & Subagents

Built-in agents: `build` and `plan` (primary, cycle with Tab); `general`, `explore`, `scout` (subagents, invoke with `@mention`); hidden system agents `title`, `summary`, `compaction`.

```jsonc
{
  "agent": {
    "build": {
      "model": "plexus/gpt-5.6-luna",
      "options": { "reasoningEffort": "max" },  // provider-specific request options
      "steps": 150,                              // max agentic iterations
      "color": "#ff5a1f",
      "permission": { "bash": { "git push *": "ask" } }
    },
    "plan": {
      "model": "plexus/gpt-5.6-terra",
      "permission": { "edit": { "*": "deny", "**/*.md": "allow" } }
    },
    "code-reviewer": {
      "description": "Reviews code for best practices",  // required for subagents
      "mode": "subagent",
      "model": "plexus/claude-sonnet-5",
      "prompt": "You are a code reviewer. Focus on security and maintainability.",
      "hidden": false
    }
  }
}
```

Markdown agents — file name (sans `.md`) becomes the agent name. Place in `~/.config/opencode/agents/` (global) or `.opencode/agents/` (project); nested paths create grouped names:

```markdown
---
description: Reviews code for quality and best practices
mode: subagent
model: anthropic/claude-sonnet-4-5
temperature: 0.1
permission:
  edit: deny
  bash: ask
---
You are in code review mode. Focus on security, performance, and maintainability.
```

Agent fields: `model`, `variant`, `prompt` (`{file:./path}` supported), `description`, `mode` (`primary`|`subagent`|`all`), `temperature`, `top_p`, `steps`, `color` (hex or theme name), `disable`, `hidden`, `options`, `permission` (merged over global; agent rules win), `tools` (deprecated — use `permission`).

Deep dive: [references/agents-commands-skills.md](references/agents-commands-skills.md)

---

## 4. Permissions & Tool Approval

```jsonc
{
  "permission": {
    "*": "allow",                       // global default for all tools
    "task": "deny",
    "bash": {                           // granular: last matching rule WINS
      "*": "allow",
      "git push *": "ask",
      "git commit *": "ask"
    },
    "edit": {
      "*": "allow",
      "~/workspace/**": "deny"
    },
    "external_directory": {
      "~/workspace/**": "allow",
      "/tmp/**": "allow"
    },
    "skill": { "*": "allow", "internal-*": "deny" }
  }
}
```

- Actions: `"allow"` (run), `"ask"` (prompt), `"deny"` (block)
- Object syntax: patterns match tool input; **last matching rule wins** — put `"*"` catch-all first, specifics after
- Wildcards: `*` (any chars), `?` (single char); `~` / `$HOME` expand in patterns
- `external_directory` gates paths outside the working directory
- Defaults: most tools `allow`; `doom_loop` and `external_directory` `ask`; `.env` reads denied
- `opencode --auto` / `opencode run --auto` auto-approves anything not explicitly denied
- Per-agent `permission` merges over global config, agent rules take precedence

Deep dive: [references/permissions.md](references/permissions.md)

---

## 5. MCP Servers

```jsonc
{
  "mcp": {
    "exa": {
      "type": "remote",
      "url": "https://plexus.home.cowger.us/mcp/exa",
      "headers": { "Authorization": "Bearer {env:AIHOME_API_KEY}" }
    },
    "github": {
      "type": "remote",
      "url": "https://plexus.home.cowger.us/mcp/github",
      "oauth": false,                    // disable OAuth auto-detection for API-key servers
      "headers": { "Authorization": "Bearer {env:AIHOME_API_KEY}" }
    },
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path"],
      "environment": { "MY_VAR": "value" },
      "timeout": 10000
    },
    "home-assistant": { "type": "remote", "url": "...", "enabled": false }  // temporarily disabled
  }
}
```

- `local` (stdio): `command` array, `environment`, `cwd`, `enabled`, `timeout`
- `remote` (Streamable HTTP): `url`, `headers`, `oauth` (`false` or `{clientId, clientSecret, scope, callbackPort, redirectUri}`), `enabled`, `timeout` (default 5000ms)
- Disable a server's tools entirely: `"tools": { "<server-name>": false }`
- Organizational remote defaults (`.well-known/opencode`) can be overridden locally with `enabled: true`

Deep dive: [references/mcp-servers.md](references/mcp-servers.md)

---

## 6. Commands, Skills & Plugins

**Commands** (slash commands) — markdown files in `~/.config/opencode/commands/` or `.opencode/commands/`, or JSON under `command`:

```markdown
---
description: Run tests with coverage
agent: build
---
Run the full test suite ($ARGUMENTS) and show failures.
Recent commits: !`git log --oneline -5`
Review @src/components/Button.tsx
```

Placeholders: `$ARGUMENTS`, `$1`–`$n`, `` !`command` `` (shell output injection), `@file` (file content injection). Options: `template`, `description`, `agent`, `model`, `variant`, `subtask`.

**Skills** — `SKILL.md` in a named folder under `.opencode/skills/`, `~/.config/opencode/skills/`, or Claude-compatible `.claude/skills/` / `~/.agents/skills/`. Frontmatter requires `name` (must match directory, `^[a-z0-9]+(-[a-z0-9]+)*$`) and `description` (1–1024 chars). Extra search paths/URLs via `skills.paths` / `skills.urls`. Gated by `permission.skill`.

**Plugins** — npm packages or local TS/JS files:

```jsonc
{
  "plugin": ["@mcowger/opencode-plexus@latest", ["./plugins/custom.ts", { "enabled": true }]]
}
```

Local files in `plugins/` (or `plugin/`) directories auto-load. Dependencies for local plugins go in `package.json` in the config directory (installed with bun at startup). Load order: global config npm plugins → project config npm plugins → global plugin dir → project plugin dir.

Deep dive: [references/agents-commands-skills.md](references/agents-commands-skills.md)

---

## 7. TUI, Themes & Keybinds (`tui.json`)

```jsonc
{
  "$schema": "https://opencode.ai/tui.json",
  "theme": "neon-fuchsia",
  "leader_timeout": 2000,
  "keybinds": {
    "leader": "ctrl+x",
    "session_new": "f1",
    "session_list": "f2",
    "editor_open": "f3",
    "model_list": "f12",
    "session_compact": "none"
  },
  "attention": { "enabled": true, "notifications": true, "sound": true, "volume": 0.4 },
  "cursor": { "style": "block", "blinking": true },
  "mouse": true,
  "scroll_speed": 3,
  "diff_style": "auto"
}
```

- Leader key (`ctrl+x` default) prefixes most bindings; `leader_timeout` default 2000ms
- Binding values: comma-separated string, array of strings, or object `{ key, event, preventDefault, fallthrough }`; `"none"` or `false` disables
- Custom themes: JSON files in `~/.config/opencode/themes/`; switch with `<leader>t`
- Legacy `theme` / `keybinds` / `tui` keys in `opencode.json` are deprecated and auto-migrate to `tui.json`

Full keybind catalog: [references/tui-keybinds.md](references/tui-keybinds.md)

---

## 8. Compaction & Context

```jsonc
{
  "compaction": {
    "auto": true,                    // compact when context is full (default true)
    "prune": false,                  // prune old tool outputs (default false)
    "tail_turns": 5,                 // recent user turns kept verbatim
    "preserve_recent_tokens": 20000, // token budget for recent-turn retention
    "reserved": 0                    // extra token buffer for compaction itself
  },
  "tool_output": {
    "max_lines": 2000,               // truncate + spill to disk above this
    "max_bytes": 51200
  }
}
```

The hidden `compaction` agent handles summarization; customize its model/prompt via `agent.compaction`.

---

## 9. Variable Substitution

Applied to config text **before** JSON parsing, in every config file:

| Syntax | Behavior |
| --- | --- |
| `{env:VAR_NAME}` | Replaced with the environment variable value (empty string if unset) |
| `{file:./relative/path}` | Replaced with the file's trimmed contents; relative to the config file's directory; `~/` supported |
| `// {file:...}` | A `{file:}` token on a line starting with `//` is left untouched (useful for JSONC comments) |

Example: `"apiKey": "{env:AIHOME_API_KEY}:opencode"`, `"prompt": "{file:./prompts/build.txt}"`.

---

## 10. Project-Local Configs

- `<repo>/opencode.json` / `opencode.jsonc` — same schema as global; safe to commit; overrides global for that repo
- `<repo>/.opencode/` — agents, commands, skills, plugins, `opencode.json(c)`, `tui.json(c)`; a `.gitignore` is auto-created for `node_modules` / lockfiles
- Discovery walks from cwd up to the git worktree; disable with `OPENCODE_DISABLE_PROJECT_CONFIG`
- Remember: arrays (e.g. `enabled_providers`) are **replaced**, not merged — a project config setting `enabled_providers` wipes the global list

---

## 11. Chezmoi Dotfile Sync & Persistence

OpenCode configuration files are versioned in git via chezmoi.

### Chezmoi Mapping

| Home Directory File | Chezmoi Source Target |
| --- | --- |
| `~/.config/opencode/opencode.jsonc` | `dot_config/opencode/opencode.jsonc` |
| `~/.config/opencode/tui.json` | `dot_config/opencode/tui.json` |
| `~/.config/opencode/agents/*.md` | `dot_config/opencode/agents/*.md` |
| `~/.config/opencode/commands/*.md` | `dot_config/opencode/commands/*.md` |
| `~/.config/opencode/skills/*/SKILL.md` | `dot_config/opencode/skills/*/SKILL.md` |
| `~/.config/opencode/themes/*.json` | `dot_config/opencode/themes/*.json` |
| `~/.config/opencode/package.json` | `dot_config/opencode/package.json` |

Do **not** chezmoi-add `~/.config/opencode/node_modules/`, `package-lock.json`, or `auth.json` — opencode/bun manages those at runtime, and `auth.json` contains secrets.

### Sync Workflow

```bash
chezmoi diff ~/.config/opencode            # Check drift
chezmoi add ~/.config/opencode/opencode.jsonc ~/.config/opencode/tui.json
chezmoi apply                              # Deploy on other machines
```

Secrets: use `{env:VAR}` substitution in config rather than committing API keys.

---

## 12. Troubleshooting & Diagnostics

| Symptom | Cause | Solution |
| --- | --- | --- |
| Custom model missing from `/models` picker | Missing `whitelist` in provider config, or provider not in `enabled_providers` | Add both; verify with `opencode models [provider]` |
| Config change not taking effect | Precedence: project/`.opencode` overrides global; array replaced wholesale | Run `opencode debug config` to see the resolved merge; check for `opencode.json` up the directory tree |
| Config parse error at startup | Invalid JSON/JSONC | Error names the file and offset; JSONC allows comments and trailing commas, but no stray commas |
| Provider auth fails / 401 | Missing or stale credentials | `opencode auth login <provider>`; check `~/.local/share/opencode/auth.json`; `{env:VAR}` may be empty |
| MCP server won't connect | Bad URL/headers, timeout, or OAuth mismatch | `opencode mcp list`; raise `timeout`; set `oauth: false` for API-key servers; check `{env:}` expansion |
| Permission rule not behaving | Last matching rule wins; agent-level rules override global | Order rules catch-all first, specifics after; inspect agent `permission` blocks |
| Keybind doesn't fire | Terminal captures the chord, or leader prefix required | Test with explicit modifier combo; check `leader` is pressed first within `leader_timeout` |
| Plugin not loading | Load order, missing dependency, or bad specifier | Check `plugin` arrays in global+project configs; ensure `package.json` deps in config dir; look for load errors on startup |
| Skill not discovered | Frontmatter/name mismatch | `SKILL.md` must be uppercase; `name` must match directory regex; check `permission.skill` isn't denying it; `opencode debug skill` |
| `enabled_providers` unexpectedly restricting models | Project config replaces the array | List the complete desired set in the highest-precedence config that defines it |
