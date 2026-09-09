---
name: omp-configuration
description: Configure and customize Oh My Pi (OMP) coding harness settings, model roles, custom providers, keybindings, MCP servers, plugins, tool approvals, and compaction. TRIGGERS - omp config, configure omp, oh my pi configuration, omp settings, model roles, omp keybindings, omp mcp, models.yml, omp plugins, omp themes, compaction settings, fallback chains.
disable-model-invocation: true
---

# OMP Configuration

A comprehensive guide and reference playbook for configuring, customizing, and troubleshooting the **Oh My Pi (OMP)** coding harness.

## When to Use This Skill

Use this skill whenever you need to:
- Configure OMP global settings (`~/.omp/agent/config.yml`) or project settings (`.omp/config.yml`)
- Assign or modify model roles (`default`, `smol`, `slow`, `plan`, `task`, `vision`, `designer`, `commit`, `advisor`)
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
| **Runtime Overrides** | CLI flags (`--model`, `--approval-mode`, `--yolo`) | Highest priority; process-local, never saved to disk |
| **CLI Overlays** | `--config <file>` or `PI_CONFIG_FILES` | Per-process YAML overlays; load in specified order |
| **Project Config** | `<cwd>/.omp/config.yml` (and `.omp/settings.json`) | Scoped strictly to current working directory `.omp/` |
| **Global Config** | `~/.omp/agent/config.yml` (or `$PI_CODING_AGENT_DIR/config.yml`) | Persistent machine-wide user configuration |
| **Built-in Defaults** | Internal settings schema | Lowest priority fallback |

### Key Config Files

| File | Path | Format | Purpose |
| --- | --- | --- | --- |
| **Main Settings** | `~/.omp/agent/config.yml` | YAML mapping | General settings, tool policies, themes, compaction |
| **Model Roles & Custom Models** | `~/.omp/agent/models.yml` | YAML mapping | Custom API endpoints, local models, provider discovery |
| **Keybindings** | `~/.omp/agent/keybindings.yml` | YAML mapping | Keyboard shortcut remaps and disables |
| **MCP Servers** | `~/.omp/agent/mcp.json` | JSON | Model Context Protocol servers (stdio, HTTP, SSE) |
| **Plugins** | `~/.omp/plugins/package.json` | JSON | Installed plugins and dependencies |
| **Plugin Settings** | `~/.omp/plugins/omp-plugins.lock.json` | JSON | Plugin status and feature configurations |

---

## Quick Reference CLI Commands

```bash
# Settings inspection & modification
omp config list                        # View all settings with current effective values
omp config list --json                 # Machine-readable JSON output
omp config get <key>                   # Inspect single key (e.g. omp config get theme.dark)
omp config set <key> <value>           # Persist a setting to global config.yml
omp config reset <key>                 # Reset key to schema default
omp config path                        # Print active agent directory path

# Models & Providers
omp models                             # List, search, and verify available models
omp usage                              # View rate limits and quotas for active providers
omp token <provider>                   # Inspect active credential / token for a provider

# Plugins & Extensions
omp plugin list                        # List installed plugins
omp plugin install <pkg>               # Install a plugin from npm / git
omp plugin remove <pkg>                # Remove an installed plugin
omp plugin search <query>              # Search plugin marketplace
```

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
  hyperlinks: auto                     # auto, on, off
display:
  shimmer: classic

# Interaction & Startup
startup:
  quiet: true
  checkUpdate: false
  changelogMode: hidden
steeringMode: all                      # all, one-at-a-time
interruptMode: wait                    # wait, immediate
autoResume: false
plan:
  enabled: true
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
| `designer` | Frontend/UI styling and design tasks | `plexus/gpt-5.6-sol:high` |
| `commit` | Commit message generation | `plexus/gemini-3.5-flash-lite:low` |
| `tiny` | Background summaries, titles, memory | `plexus/gemini-3.5-flash-lite:low` |

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
  designer: plexus/gpt-5.6-sol:high
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
  maxDelayMs: 300000                   # 5 minutes ceiling
  modelFallback: true
  fallbackRevertPolicy: cooldown-expiry # cooldown-expiry or never
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
    # Model-specific wildcard fallback (swaps provider)
    google-antigravity/*:
      - google/*
      - google-vertex/*
```

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
- `app.retry` (`Alt+R`)
- `app.agents.hub` (`Alt+A`)
- `app.live.toggle` (`Ctrl+L`)

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
- **`stdio`**: Local subprocess. Requires `command`, optional `args`, `env`, `cwd`.
- **`sse`**: Server-Sent Events (legacy remote). Requires `url`, optional `headers`.

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
- `anthropic-messages` (Anthropic Messages API `/v1/messages`)
- `google-vertex` / `google-generative-ai`

---

## 7. Tool Approvals, Bash Patterns & Interceptor

Fine-tune safety, permissions, and tool routing.

```yaml
tools:
  format: auto                         # auto, native, xml, anthropic, etc.
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
  direnv: "off"
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

---

## 8. Compaction, Memory & Context

Control conversation lifespan and long-term memory.

```yaml
compaction:
  enabled: true
  thresholdPercent: 70                 # Trigger when context reaches 70%
  idleEnabled: true
  keepRecentTokens: 20000
  methodOrder:
    - remote                           # Provider server-side compaction
    - snapcompact                      # Visual screenshot compaction
    - handoff                          # Structured continuation summary
    - shake                            # Prune stale tool calls
    - soft                             # Truncate oldest turns
snapcompact:
  shape: auto

# Long-term Memory
memory:
  backend: local                       # off, local, hindsight, mnemopi

# Automatic Learning & Skill Minting
autolearn:
  enabled: true
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
Isolate environments using profiles:
```bash
omp --profile work                     # Runs with ~/.omp/profiles/work/agent/
omp --alias work                       # Creates shell alias 'omp-work'
```

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
| `Unknown setting` from `omp config set` | Key path is incorrect or uses shorthand | Check `omp config list` for exact dotted schema path (e.g. `theme.dark`, not `theme`) |
| Project setting not taking effect | Working directory lacks `.omp/config.yml` | Ensure `.omp/` is in the cwd where `omp` was launched (discovery does not check parent directories) |
| Array in project wiped out global items | Arrays replace rather than append | In project `.omp/config.yml`, list the complete desired array items |
| Model fallback fails or loops | Fallback chain contains invalid/unreachable models | Check `retry.fallbackChains` and verify models with `omp models` |
| MCP server fails to connect | Missing dependencies, bad URL, or timeout | Run `omp mcp test <name>` or verify URL/token; set `OMP_MCP_TIMEOUT_MS=0` to test without timeout |
| Keybinding doesn't fire | Terminal captures chord or unqualified name | Use exact namespaced action ID (e.g. `app.model.selectTemporary`) and test alternative chord |
| Config corruption / crash at startup | Invalid YAML mapping | OMP saves backup to `.broken-*`; check YAML syntax with `python -c 'import yaml; yaml.safe_load(open("config.yml"))'` |
