---
name: omp-configuration
description: Configure and customize Oh My Pi (OMP) coding harness settings, model roles, custom providers, profiles, keybindings, MCP servers, plugins, tool approvals, advisor, and compaction. TRIGGERS - omp config, configure omp, oh my pi configuration, omp settings, model roles, omp profiles, omp keybindings, omp mcp, models.yml, omp plugins, omp themes, compaction settings, fallback chains, vim mode.
disable-model-invocation: true
---

# OMP Configuration

A comprehensive guide and reference playbook for configuring, customizing, and troubleshooting the **Oh My Pi (OMP)** coding harness.

## When to Use This Skill

Use this skill whenever you need to:
- Configure OMP global settings (`~/.omp/agent/config.yml`) or project settings (`.omp/config.yml`)
- Assign or modify model roles (`default`, `smol`, `slow`, `vision`, `plan`, `commit`, `tiny`, `task`, `advisor`) and custom roles
- Define custom providers or local models (Ollama, vLLM, LM Studio, LiteLLM, OpenAI-compatible) in `models.yml`
- Remap or disable keyboard shortcuts in `~/.omp/agent/keybindings.yml`
- Add, update, or troubleshoot MCP servers in `mcp.json` (stdio, Streamable HTTP, SSE)
- Tune tool approval policies, bash pattern rules, or bash interceptor redirects
- Adjust context compaction methods (`remote`, `snapcompact`, `handoff`, `shake`, `soft`), thresholds, or memory backends
- Install and manage OMP extensions and plugins (`~/.omp/plugins/`)
- Sync and backup OMP configurations across machines using chezmoi

---

## Architecture & Configuration Hierarchy

| Scope | Location | Purpose & Precedence |
| --- | --- | --- |
| **Runtime Overrides** | CLI flags (`--model`, `--approval-mode`, `--yolo`) and feature env vars | Highest priority; process-local, never saved to disk |
| **CLI Overlays** | `--config <file>` or `PI_CONFIG_FILES` | Per-process YAML overlays; load in specified order |
| **Project Config** | `<cwd>/.omp/config.yml` (merged over `<cwd>/.omp/settings.json`) | Scoped strictly to current working directory `.omp/` |
| **Global Config** | `~/.omp/agent/config.yml` (or existing `config.yaml`; `$PI_CODING_AGENT_DIR` relocates the base) | Persistent machine-wide user configuration |
| **Built-in Defaults** | Settings schema | Lowest priority fallback |

Settings discovery checks only the process working directory's `.omp/` — it does **not** walk ancestor directories. A `--config` overlay with a missing file, invalid YAML, or a top-level array/scalar is a hard error (no silent fallback). An invalid persistent settings file is moved to a `.broken-*` backup on writable startup.

### Key Config Files

| File | Path | Format | Purpose |
| --- | --- | --- | --- |
| **Main Settings** | `~/.omp/agent/config.yml` | YAML mapping | General settings, tool policies, themes, compaction |
| **Model Roles & Custom Models** | `~/.omp/agent/models.yml` | YAML mapping | Custom API endpoints, local models, provider discovery |
| **Keybindings** | `~/.omp/agent/keybindings.yml` | YAML mapping | Keyboard shortcut remaps and disables |
| **MCP Servers** | `~/.omp/agent/mcp.json` | JSON | Model Context Protocol servers (stdio, HTTP, SSE) |
| **Plugins** | `~/.omp/plugins/package.json` | JSON | Installed plugins and dependencies |
| **Plugin Settings** | `~/.omp/plugins/omp-plugins.lock.json` | JSON | Plugin status and feature configurations |
| **Profiles** | `~/.omp/profiles/<name>/agent/` | Directory | Isolated auth, settings, models, MCP, sessions, caches |

---

## Quick Reference CLI Commands

```bash
# Settings inspection & modification
omp config list                        # View all settings with current effective values
omp config list --json                 # Machine-readable JSON output
omp config get <key>                   # Inspect single key (e.g. omp config get theme.dark)
omp config set <key> <value>           # Persist a setting to global config.yml
omp config reset <key>                 # Write the schema DEFAULT back into config.yml
omp config path                        # Print active agent directory path
omp config init-xdg                    # Create omp dirs under XDG data/state/cache (Linux/macOS)

# Models, providers & usage
omp models [ls|find <q>|refresh]       # List, search, or refresh available models
omp usage                              # Provider rate limits/quotas (subcommands: clients, invalidate)
omp token <provider>                   # Print the API key / OAuth token for a provider

# Plugins & extensions
omp plugin list                        # List installed plugins
omp plugin install <pkg>[features]     # Install from npm, git, or a local path
omp plugin uninstall <pkg>             # Remove a plugin
omp plugin upgrade [pkg@mkt]           # Upgrade plugins
omp plugin enable|disable <pkg>        # Toggle a plugin
omp plugin features <pkg>              # View/modify enabled features
omp plugin config <list|get|set> ...   # Manage plugin settings
omp plugin marketplace <add|remove|update|list>
omp plugin discover [marketplace]      # Browse marketplace plugins
omp plugin doctor [--fix]              # Check plugin health

# Other useful subcommands
omp agents [unpack]                    # Manage bundled task agents
omp worktree [wt]                      # List/clear agent-managed git worktrees
omp search [q]                         # Test web search providers from the CLI
omp stats                              # View usage statistics
omp ps                                 # List/control daemon background processes
```

Run `omp --help` or `omp <command> --help` for the full surface. MCP is managed via slash commands inside a session (`/mcp list`, `/mcp test <name>`, `/mcp reload`, …), not a CLI subcommand.

---

## 1. Main Settings (`config.yml`)

Settings use nested YAML mappings. Dotted paths map 1:1 to nested objects.

### Merge Rules
- **Objects are deep-merged**: Keys present in lower layers are preserved unless overridden.
- **Arrays & Scalars are replaced wholesale**: Higher precedence layer arrays **completely replace** lower layer arrays. Never expect arrays to append automatically.

### Essential Configuration Example

```yaml
# ~/.omp/agent/config.yml
setupVersion: 2

# Appearance & Interface
theme:
  dark: titanium
  light: light
symbolPreset: nerd                     # unicode, nerd, ascii
terminal:
  showProgress: true
tui:
  textSizing: false
  hyperlinks: auto                     # off, auto, always
  vimMode: false                       # modal prompt editing
display:
  shimmer: classic                     # classic, kitt, disabled

# Interaction & Startup
startup:
  quiet: true
  checkUpdate: false
  changelogMode: hidden                # summary, expanded, hidden
steeringMode: all                      # all, one-at-a-time
followUpMode: one-at-a-time            # all, one-at-a-time
interruptMode: wait                    # wait, immediate
autoResume: false
composer:
  recallClearedDrafts: true            # Ctrl+C clears a draft; Up recalls it
plan:
  enabled: true
  autosave: false                      # autosave approved plans to .omp/plans/
todo:
  enabled: true

# Thinking & Reasoning
defaultThinkingLevel: high             # minimal, low, medium, high, xhigh, max, auto
proseOnlyThinking: true
hideThinkingBlock: false
```

---

## 2. Model Roles, Routing & Thinking

Model roles define which model handles specific categories of work.

### Built-in Roles

| Role | Intended Workload | Example Assignment |
| --- | --- | --- |
| `default` | Primary interactive conversational partner | `plexus/claude-sonnet-5` |
| `task` | Autonomous task execution, subagents | `plexus/gpt-5.6-luna:high` |
| `smol` | Fast, lightweight queries, simple transforms | `plexus/gemini-3.5-flash-lite:low` |
| `slow` | Complex reasoning, deep architectural analysis | `plexus/claude-opus-5:high` |
| `plan` | Plan mode drafting and review | `plexus/claude-opus-5:high` |
| `advisor` | Background turn review and watchdog | `plexus/claude-opus-5:high` |
| `vision` | Image inspection and UI analysis | `plexus/gpt-5.6-terra:medium` |
| `commit` | Commit message generation | `plexus/gemini-3.5-flash-lite:low` |
| `tiny` | Background summaries, titles, memory | `plexus/gemini-3.5-flash-lite:low` |

Built-in roles are exactly `default`, `smol`, `slow`, `vision`, `plan`, `commit`, `tiny`, `task`, `advisor`. Custom roles are allowed — any role name assigned in `modelRoles` or declared in `modelTags` becomes selectable. Role selectors: `@<role>`, `*` (default), and the legacy `pi/<role>` prefix.

### Role Configuration & Thinking Suffix

Append `:level` to any model identifier to set explicit reasoning effort:
- Suffixes: `:minimal`, `:low`, `:medium`, `:high`, `:xhigh`, `:max`

```yaml
modelRoles:
  default: plexus/claude-sonnet-5
  task: plexus/gpt-5.6-luna:high
  smol: plexus/gemini-3.5-flash-lite:low
  slow: plexus/claude-opus-5:high
  plan: plexus/claude-opus-5:high
  advisor: plexus/claude-opus-5:high
  vision: plexus/gpt-5.6-terra:medium
  commit: plexus/gemini-3.5-flash-lite:low
  tiny: plexus/gemini-3.5-flash-lite:low

cycleOrder:
  - smol
  - default
  - slow

# Restrict or filter models
enabledModels:
  - plexus/*

# Disable unused providers / discovery sources
disabledProviders:
  - openai
  - anthropic
  - google
  - openrouter
  - groq
```

---

## 3. Retry Logic & Fallback Chains

Configure automatic failover when a provider hits rate limits (429), quota walls, or transient outages.

```yaml
retry:
  enabled: true
  maxRetries: 10
  baseDelayMs: 500
  maxDelayMs: 300000                   # 5 minutes ceiling (0 disables the cap)
  modelFallback: true
  fallbackRevertPolicy: cooldown-expiry # cooldown-expiry or never
  waitForUsageReset: false             # sleep until a provider-stated quota reset
  usageAwareFallback: false            # fall back before a coding-plan model hard-fails
  usageReservePct: 10                  # "near its limit" remaining-% threshold
  fallbackChains:
    # Role-based fallback
    default:
      - plexus/gpt-5.6-terra
      - plexus/claude-sonnet-5
    task:
      - plexus/gpt-5.6-terra
    smol:
      - plexus/gemini-3.5-flash-lite
    slow:
      - plexus/claude-opus-5
      - plexus/claude-sonnet-5
    plan:
      - plexus/claude-opus-5
      - plexus/claude-sonnet-5
    # Model-specific wildcard fallback (swaps provider, keeps the model id)
    google-antigravity/*:
      - google/*
      - google-vertex/*
```

**Chain resolution.** When the active model keeps failing, the session picks the chain that owns it by specificity: an exact `provider/model-id` key, then a `provider/*` wildcard, then the current role's chain, then `default`. A key containing `/` is model-oriented and wins over roles; a `provider/*` **entry** keeps the failing model's id and swaps the provider. Subagents get their own per-spawn chains when their agent definition lists multiple model patterns.

---

## 4. Keybindings Configuration (`keybindings.yml`)

User keyboard shortcuts live in `~/.omp/agent/keybindings.yml`.

### Action Remap Syntax

```yaml
# ~/.omp/agent/keybindings.yml
app.model.selectTemporary:
  - F12
  - Alt+P
app.model.select:
  - Alt+M
app.thinking.cycle:
  - F11
  - Shift+Tab
app.plan.toggle:
  - Alt+Shift+P
app.history.search:
  - Ctrl+R
app.agents.hub:
  - Alt+A

# Disable a keybinding completely with empty array
app.editor.external: []
```

### Common Action IDs

- `app.model.cycleForward` / `app.model.cycleBackward` (`Ctrl+P` / `Shift+Ctrl+P`)
- `app.model.selectTemporary` (`Alt+P`)
- `app.model.select` (`Alt+M`)
- `app.thinking.cycle` (`Shift+Tab`)
- `app.thinking.toggle` (`Ctrl+T`)
- `app.plan.toggle` (`Alt+Shift+P`)
- `app.history.search` (`Ctrl+R`)
- `app.tools.expand` (`Ctrl+O`)
- `app.tools.toggleVisibility` (`Ctrl+Shift+O`)
- `app.message.followUp` (`Ctrl+Q`, `Ctrl+Enter`)
- `app.message.dequeue` (`Alt+Up`, `Shift+Up`)
- `app.retry` (`F5`, `Alt+R`)
- `app.interrupt` (`Escape`), `app.clear` (`Ctrl+C`), `app.exit` (`Ctrl+D`)
- `app.agents.hub` (`Alt+A`)
- `app.live.toggle` (`Ctrl+L`)
- `app.display.reset` (`Alt+L`)
- `app.clipboard.copyLine` (`Alt+Shift+L`), `app.clipboard.copyPrompt` (`Alt+Shift+C`)

TUI actions are namespaced under `tui.*` (`tui.editor.*`, `tui.input.*`, `tui.select.*`) and remap the same way. See `references/keybindings.md` for the full catalog and Vim-mode keys.

---

## 5. MCP Servers Configuration (`mcp.json`)

Configure external tool servers via Model Context Protocol in `~/.omp/agent/mcp.json` (user) or `.omp/mcp.json` (project).

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "exa": {
      "type": "http",
      "url": "https://plexus.home.cowger.us/mcp/exa",
      "headers": {
        "Authorization": "Bearer sk-SuperSecretValue",
        "Accept": "application/json, text/event-stream"
      }
    },
    "github": {
      "type": "http",
      "url": "https://plexus.home.cowger.us/mcp/github",
      "headers": {
        "Authorization": "Bearer sk-SuperSecretValue"
      }
    },
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/matt.cowger/workspace"]
    },
    "home-assistant": {
      "type": "http",
      "url": "https://plexus.home.cowger.us/mcp/home-assistant",
      "headers": {
        "Authorization": "Bearer sk-SuperSecretValue"
      },
      "enabled": false
    }
  },
  "disabledServers": []
}
```

### Transport Types
- **`http`** (Streamable HTTP): Recommended for remote/proxy servers. Requires `url`, optional `headers`.
- **`stdio`**: Local subprocess. Default when `type` is omitted. Requires `command`, optional `args`, `env`, `cwd`.
- **`sse`**: Server-Sent Events (legacy remote). Requires `url`, optional `headers`.

### Shared Server Fields
- `enabled` — skip when `false` (unless listed in the user `enabledServers` allowlist).
- `timeout` — per-server request timeout in ms; `0` disables client-side timeouts. `OMP_MCP_TIMEOUT_MS` overrides all server timeouts process-wide.
- `requestIdFormat` — `"number"` (default) or `"string"` (collision-resistant snowflake ids); OMP-native files only.
- `oauth` / `auth` — explicit OAuth client settings and stored-credential metadata.

### Session Commands
MCP is managed via slash commands, not a CLI subcommand: `/mcp add`, `/mcp list`, `/mcp test <name>`, `/mcp reload`, `/mcp reconnect <name>`, `/mcp reauth <name>` / `/mcp unauth <name>`, `/mcp enable|disable <name>`, `/mcp resources`, `/mcp prompts`, `/mcp notifications`. Project config loading is governed by `mcp.enableProjectConfig` (default `true`).

---

## 6. Custom Models & Providers (`models.yml`)

Configure custom LLM backends in `~/.omp/agent/models.yml`.

### Example Configurations

```yaml
providers:
  # Local Ollama
  ollama:
    baseUrl: http://127.0.0.1:11434
    api: openai-responses
    auth: none
    discovery:
      type: ollama

  # Local vLLM / LM Studio / OpenAI-Compatible Server
  local-vllm:
    baseUrl: http://127.0.0.1:8000/v1
    api: openai-completions
    apiKey: "none"
    models:
      - id: Qwen/Qwen2.5-Coder-32B-Instruct
        name: Qwen 2.5 Coder 32B
        contextWindow: 65536
        maxTokens: 8192
        reasoning: false
        input: [text]

  # LiteLLM Proxy with Discovery
  litellm-proxy:
    baseUrl: http://127.0.0.1:4000/v1
    apiKey: "!op read op://dev/litellm/api-key"   # Command-resolved secret
    api: openai-completions
    discovery:
      type: litellm

  # Plexus Gateway / Custom Endpoint
  plexus:
    baseUrl: https://plexus.home.cowger.us/v1
    apiKey: "!echo $AIHOME_API_KEY"
    api: openai-responses
```

### Supported API Types
- `openai-completions` (Chat Completions `/v1/chat/completions`)
- `openai-responses` (OpenAI Responses `/v1/responses`)
- `openai-codex-responses` (OpenAI Codex)
- `azure-openai-responses` (Azure OpenAI)
- `anthropic-messages` (Anthropic Messages API `/v1/messages`)
- `bedrock-converse-stream` (AWS Bedrock Converse Stream)
- `google-generative-ai` (Google AI Studio Gemini API)
- `google-gemini-cli` (Gemini CLI OAuth surface)
- `google-vertex` (Vertex AI)

### Other Provider Fields
- `auth`: `apiKey` (default), `none`, or `oauth`.
- `discovery.type`: `ollama`, `llama.cpp`, `lm-studio`, `litellm`, `openai-models-list`, or `proxy`; optional `timeoutMs` and (for `openai-models-list`) `injectV1`.
- `transport: pi-native` dispatches every model through an auth-gateway `POST /v1/pi/stream`.
- Per-model: `tokenizer`, `imageInputDecoder: stb`, `thinking`, `compat`, `contextPromotionTarget`, `compactionModel`, `remoteCompaction`, and Bedrock `guardrail*`/`requestMetadata` at provider level.

See `references/models-providers.md` for the complete field and `compat` catalog.

---

## 7. Tool Approvals, Bash Patterns & Interceptor

Fine-tune safety, permissions, and tool routing.

```yaml
tools:
  format: auto                         # auto, native, glm, hermes, kimi, xml, anthropic, deepseek, harmony, qwen3, gemini, gemma, minimax
  approvalMode: yolo                   # yolo (auto-approve all), write (reads+writes), always-ask
  approval:
    bash: allow
    edit: allow
    read: allow
    eval: allow
  intentTracing: true
  maxTimeout: 0                        # 0 = no timeout

# Bash pattern rules (first match wins)
bash:
  direnv: auto                         # auto loads .envrc; off skips it
  allowCompoundCommands: false         # opt-in: evaluate flat literal && chains per segment
  autoBackground:
    enabled: true
    thresholdMs: 60000
  patterns:
    - match: "git push*"
      approval: prompt
    - match: "rm -rf /*"
      approval: deny
    - match: "*"
      approval: allow

# Bash Interceptor (redirects shell commands to dedicated tools)
bashInterceptor:
  enabled: true
  patterns:
    - pattern: '^\s*(cat|head|tail)\s+'
      tool: read
      message: "Use the read tool instead."
```

`bash.patterns` is an approval policy, not containment — an allowed program keeps full shell access. It governs the `bash` tool only; also set `tools.approval.eval` to cover shells started through `eval`.

---

## 8. Compaction, Memory & Context

Control conversation lifespan and long-term memory.

```yaml
compaction:
  enabled: true
  methodOrder:                         # fallback order of strategies
    - remote                           # Provider server-side compaction
    - snapcompact                      # Visual screenshot compaction
    - handoff                          # Structured continuation summary
    - shake                            # Prune stale tool calls
    - soft                             # Truncate oldest turns
  thresholdPercent: -1                 # -1 = reserve-based; else % of context window
  thresholdTokens: -1                  # -1 = use percentage/reserve
  keepRecentTokens: 20000
  midTurnEnabled: true                 # check at safe mid-turn tool-loop boundaries
  autoContinue: true
  idleEnabled: false                   # compact while idle
  idleThresholdTokens: 200000
  idleTimeoutSeconds: 300
  experimentalContextManagement: false # notes-backed context windows (experimental)
snapcompact:
  shape: auto                          # auto or a frame variant (8on22-bw, 11on16-bw, silver16-bw, doc-8on16-bw, …)

# Context promotion / extended windows
contextPromotion:
  enabled: false                       # promote to a larger-context model instead of compacting
extendedContext: false                 # use premium long-context tiers

# Long-term Memory
memory:
  backend: off                         # off, local, hindsight, mnemopi, sharpshooter

# Automatic Learning & Skill Minting
autolearn:
  enabled: false
  autoContinue: false
  minToolCalls: 5
```

---

## 9. Plugins & Extensions

OMP extensions package custom skills, hooks, tools, and MCP servers.

### Config in `config.yml`
```yaml
extensions:
  - /home/matt.cowger/.omp/plugins/node_modules/@mcowger/oh-my-pi-plexus/dist/extension.js
```

### Plugin Manifest (`~/.omp/plugins/package.json`)
```json
{
  "name": "omp-plugins",
  "private": true,
  "dependencies": {
    "@mcowger/oh-my-pi-plexus": "1.4.7"
  }
}
```

Plugins can also be installed at project scope (`<project>/.omp/plugins`, via `omp plugin install --scope project name@marketplace`), where they shadow same-named user plugins. Runtime state and per-plugin settings live in `omp-plugins.lock.json` under the same root.

### Extension Configuration
Individual extensions store local options under `~/.omp/agent/extensions/<name>/config.json`.
Example (`~/.omp/agent/extensions/plexus/config.json`):
```json
{
  "baseUrl": "https://plexus.home.cowger.us"
}
```

---

## 10. Project-Local & Profile-Specific Configs

### Project Settings (`<repo>/.omp/config.yml`)
Create a `.omp/config.yml` in a repository to override settings specifically for that repository:
```yaml
# <repo>/.omp/config.yml
modelRoles:
  default: anthropic/claude-sonnet-4-5
  task: openai/gpt-5.6-luna:high

tools:
  approvalMode: write
  approval:
    bash: prompt

compaction:
  thresholdPercent: 80
```
*Note: Project arrays (like `disabledProviders`) replace global arrays completely.*

### Named Profiles
Isolate environments (auth, sessions, settings, models, MCP, caches) using profiles:
```bash
omp --profile work                     # Runs with ~/.omp/profiles/work/agent/
omp --alias work                       # Creates a shell shortcut for the selected profile
```

`--alias` requires `--profile <name>` (or `OMP_PROFILE`/`PI_PROFILE`); it writes a shell function (bash/zsh/fish/pwsh) of the form `<alias>() { command omp --profile=<name> "$@"; }` and exits. Project-scoped config is keyed to the working directory and applies under every profile.

---

## 11. Chezmoi Dotfile Sync & Persistence

OMP configuration files are versioned in git via chezmoi.

### Chezmoi Mapping

| Home Directory File | Chezmoi Source Target |
| --- | --- |
| `~/.omp/agent/config.yml` | `dot_omp/private_agent/private_config.yml` |
| `~/.omp/agent/keybindings.yml` | `dot_omp/private_agent/keybindings.yml` |
| `~/.omp/agent/mcp.json` | `dot_omp/private_agent/private_mcp.json` |
| `~/.omp/agent/models.yml` | `dot_omp/private_agent/models.yml` |
| `~/.omp/agent/extensions/plexus/config.json` | `dot_omp/private_agent/extensions/plexus/config.json` |
| `~/.omp/plugins/package.json` | `dot_omp/plugins/package.json` |
| `~/.omp/plugins/omp-plugins.lock.json` | `dot_omp/plugins/omp-plugins.lock.json` |

### Sync Workflow
```bash
# Check drift between ~/.omp and chezmoi repo
chezmoi diff ~/.omp

# Add modified configs
chezmoi add ~/.omp/agent/config.yml ~/.omp/agent/keybindings.yml ~/.omp/agent/mcp.json

# Commit & push (if not auto-pushed)
chezmoi git -- push
```

---

## 12. Troubleshooting & Diagnostics

| Symptom | Cause | Solution |
| --- | --- | --- |
| `Unknown setting` from `omp config set` | Key path is incorrect or uses shorthand | Check `omp config list` for the exact dotted schema path (e.g. `theme.dark`, not `theme`) |
| Project setting not taking effect | Working directory lacks a non-empty `.omp/config.yml` | Start `omp` from the directory containing `.omp/` (discovery does not check parent directories) |
| Array in project wiped out global items | Arrays replace rather than append | In project `.omp/config.yml`, list the complete desired array items |
| `omp config reset` did not remove my key | `reset` persists the schema default, it does not delete | Delete the key from `~/.omp/agent/config.yml` by hand |
| Model fallback fails or loops | Chain contains invalid/unreachable models | Check `retry.fallbackChains`; unknown models/providers warn at startup; verify with `omp models` |
| Provider still available after disabling | Provider id vs discovery-source id mismatch | `disabledProviders` gates both namespaces (`anthropic` = model backend, `claude` = Claude-format discovery); a project array may be replacing the global one |
| MCP server fails to connect | Missing dependency, bad URL, or timeout | Run `/mcp test <name>`; verify URL/token; set `OMP_MCP_TIMEOUT_MS=0` to test without a timeout |
| Keybinding doesn't fire | Terminal captures the chord, or a legacy name is used | Use the exact namespaced action ID (e.g. `app.model.selectTemporary`); test an alternative chord |
| `--config` overlay fails at startup | Missing file, invalid YAML, or top-level array/scalar | Overlays are hard errors with no silent fallback; fix the path or contents |
| Env var beats my config | Feature env vars/flags override `config.yml` | Unset the variable or drop the flag to let the persisted value win |
| Config corruption / crash at startup | Invalid YAML mapping | OMP moves the file to a `.broken-*` backup; check YAML with `python -c 'import yaml; yaml.safe_load(open("config.yml"))'` |
