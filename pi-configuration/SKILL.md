---
name: pi-configuration
description: Configure and customize the Pi coding agent (@earendil-works/pi-coding-agent) settings, custom providers and models, keybindings, themes, extensions, packages, skills, prompt templates, and MCP servers via the pi-mcp-adapter plugin. TRIGGERS - pi config, configure pi, pi settings, settings.json, pi models, models.json, custom provider, pi keybindings, keybindings.json, pi themes, pi extensions, pi packages, pi install, pi skills, prompt templates, PI_CODING_AGENT_DIR, pi mcp, pi-mcp-adapter, .mcp.json, directTools.
disable-model-invocation: true
---

# Pi Configuration

A comprehensive guide and reference playbook for configuring, customizing, and troubleshooting the **Pi** coding agent (`@earendil-works/pi-coding-agent`, earendil-works/pi fork of badlogic/pi-mono), including the **pi-mcp-adapter** plugin for MCP servers.

## When to Use This Skill

Use this skill whenever you need to:
- Configure Pi global settings (`~/.pi/agent/settings.json`) or project settings (`.pi/settings.json`)
- Set the default provider/model, thinking levels, and which models appear in Ctrl+P cycling
- Add custom providers or local models (Ollama, vLLM, LM Studio[deps], any OpenAI-compatible endpoint) in `models.json`
- Remap keyboard shortcuts in `~/.pi/agent/keybindings.json`
- Create or install themes (`~/.pi/agent/themes/*.json`)
- Install packages and extensions (`pi install npm:...`, `~/.pi/agent/extensions/`)
- Add skills (`~/.pi/agent/skills/`, `.agents/skills/`) or prompt templates (`~/.pi/agent/prompts/`)
- Configure MCP servers — **core Pi has no native MCP support**; use the `pi-mcp-adapter` package (see section 8)
- Understand project trust (`trust.json`, `--approve`/`--no-approve`)
- Tune compaction, retry, terminal rendering, or proxy settings
- Troubleshoot config issues (`/settings`, `/reload`, `/hotkeys`)

---

## Architecture & Configuration Hierarchy

| Scope | Location | Purpose & Precedence |
| --- | --- | --- |
| **Runtime / CLI flags** | `pi --model ...`, `--tools ...`, `--session-dir ...` | Highest priority; process-local |
| **Project settings** | `<repo>/.pi/settings.json` (+ `.pi/extensions|skills|prompts|themes/`, `.pi/SYSTEM.md`, `.pi/mcp.json`) | Overrides global; gated by project trust |
| **Global settings** | `~/.pi/agent/settings.json` (or `$PI_CODING_AGENT_DIR/`) | Persistent machine-wide user configuration |
| **Built-in defaults** | Internal settings schema | Lowest priority fallback |

**Merge rules:** nested objects are deep-merged; **arrays and scalars are replaced wholesale** (a project `defaultTools` replaces the global array — never appended).

**Project trust:** `.pi/` resources (settings, extensions, skills, prompts, themes, SYSTEM.md, packages) only load when the directory is trusted. Trust decisions persist in `~/.pi/agent/trust.json` and inherit from the closest trusted parent. Policy via `defaultProjectTrust`: `ask` (default) | `always` | `never`; per-run override `--approve` / `--no-approve`. Saved via `/trust`. Non-interactive modes never prompt. Trust is an input-loading guard, **not a sandbox**.

### Key Config Files

| File | Path | Format | Purpose |
| --- | --- | --- | --- |
| **Main Settings** | `~/.pi/agent/settings.json` | JSON (strict, no comments) | Models, theme, compaction, terminal, resources |
| **Custom Models/Providers** | `~/.pi/agent/models.json` | JSON | Custom providers, local models, provider overrides |
| **Credentials** | `~/.pi/agent/auth.json` | JSON (mode 0600) | Provider API keys / OAuth; via `/login`, env vars, or file |
| **Keybindings** | `~/.pi/agent/keybindings.json` | JSON | Key remaps (global only) |
| **Trust** | `~/.pi/agent/trust.json` | JSON | Per-directory project trust decisions |
| **Extensions** | `~/.pi/agent/extensions/` | TS/JS | Auto-loaded extension files |
| **Skills** | `~/.pi/agent/skills/`, `~/.agents/skills/` | SKILL.md | Agent skills (see section 7) |
| **Prompts** | `~/.pi/agent/prompts/*.md` | Markdown | Slash-command prompt templates |
| **Themes** | `~/.pi/agent/themes/*.json` | JSON | Custom TUI themes |
| **Packages** | `~/.pi/agent/npm/`, `~/.pi/agent/git/` | — | Installed npm/git packages |
| **Sessions** | `~/.pi/agent/sessions/` | JSONL | Session transcripts |
| **MCP config** | `~/.pi/agent/mcp.json`, `.mcp.json`, `.pi/mcp.json` | JSON | pi-mcp-adapter servers & settings |

Project equivalents live under `<repo>/.pi/` (`settings.json`, `extensions/`, `skills/`, `prompts/`, `themes/`, `SYSTEM.md`, `APPEND_SYSTEM.md`, `npm/`, `git/`, `mcp.json`) — **except** `models.json`, which is global-only; project-level providers come from extensions/packages.

Paths inside settings files resolve relative to their containing config dir (`~/.pi/agent/` or `.pi/`); `~` and absolute paths supported. `PI_CODING_AGENT_DIR` overrides the global root and `PI_CODING_AGENT_SESSION_DIR` the session dir.

---

## Quick Reference CLI Commands

```bash
pi                                   # Start in cwd; pi --continue / --resume / --session <id>
pi --provider openai --model gpt-4o  # Startup overrides (also --api-key, --thinking)
pi --model sonnet:high               # Name alias + thinking suffix
pi --models "claude-*,gpt-4o"        # Restrict Ctrl+P cycle list
pi --list-models [filter]            # List available models

# Packages & extensions
pi install npm:pi-mcp-adapter        # Install package (npm:, git:, https://, or local path)
pi install -l npm:@foo/bar           # Project-local install (.pi/npm/)
pi remove npm:@foo/bar               # Uninstall
pi list                              # List installed packages
pi update [--all|--extensions|--models|--self [pkg]]   # Update packages/self
pi -e npm:@foo/bar                   # Temporarily load a package/extension this run
pi config [-l]                       # Edit package/resource config (-l = project)

# Resources & tools (per-run)
pi -e ./ext.ts        pi --skill ./s/SKILL.md      pi --prompt-template ./review.md
pi --no-extensions    pi --no-skills                pi --no-prompt-templates
pi --tools read,grep,ls   pi --exclude-tools bash   pi --no-builtin-tools   pi --no-tools

# Trust
pi --approve / pi --no-approve       # Alias -a / -na; trust project config for this run
```

Key interactive commands: `/settings` (edit common settings), `/model` (Ctrl+S saves default), `/thinking` (Ctrl+S saves), `/scoped-models`, `/login` `/logout`, `/trust`, `/reload` (reload keybindings, extensions, skills, prompts, themes, context files), `/hotkeys` (effective keybindings), `/share`, `/tree`.

---

## 1. Main Settings (`settings.json`)

Strict JSON at `~/.pi/agent/settings.json` (global) and `.pi/settings.json` (project). Common settings — the full catalog with defaults is in [references/settings.md](references/settings.md):

```json
{
  "defaultProvider": "plexus",
  "defaultModel": "muse-spark-1.3",
  "defaultThinkingLevel": "max",
  "modelThinkingLevels": { "plexus/gpt-5.6-luna": "max" },
  "enabledModels": ["claude-*", "gpt-4o"],

  "theme": "breezy-ocean",
  "tuiMode": "fullscreen",
  "fullscreenExitOutput": "resume-hint",
  "quietStartup": true,
  "collapseChangelog": true,
  "enableInstallTelemetry": false,
  "defaultProjectTrust": "always",
  "doubleEscapeAction": "none",

  "showCacheMissNotices": true,
  "hideThinkingBlock": false,
  "externalEditor": "code --wait",
  "shellCommandPrefix": "shopt -s expand_aliases",

  "compaction": { "enabled": true, "reserveTokens": 16384, "keepRecentTokens": 20000 },
  "retry": { "enabled": true, "maxRetries": 3, "baseDelayMs": 2000 },
  "terminal": { "showTerminalProgress": true, "clearOnShrink": false, "hyperlinks": "auto" },
  "images": { "autoResize": true, "blockImages": false },
  "defaultTools": ["read", "bash", "edit", "write", "grep", "find", "ls"],
  "enableSkillCommands": true,
  "packages": ["npm:pi-mcp-adapter"]
}
```

Notables:
- Thinking levels: `off | minimal | low | medium | high | xhigh | max`; custom budgets via `thinkingBudgets`.
- `enabledModels` filters which models appear in the Ctrl+P cycle list (glob support).
- `defaultTools` selects built-in tools (`read bash powershell edit write grep find ls`); `[]` disables built-ins but keeps extension tools. CLI `--tools` is a stricter allowlist.
- `httpProxy` (global only), `transport` (`sse|websocket|websocket-cached|auto`), `httpIdleTimeoutMs`, `sessionDir`.
- `steeringMode` / `followUpMode`: `all | one-at-a-time`.
- Unknown keys are harmless — extensions may add their own top-level settings (this machine's `enabledProviders` and `tools` keys come from the `@mcowger/pi-suppress-providers` extension, not core Pi).

---

## 2. Custom Providers & Models (`models.json`)

Global-only file: `~/.pi/agent/models.json`. Reloads whenever `/model` opens. Supported APIs: `openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai` (more exotic types like `google-vertex`, `bedrock-converse-stream`, `azure-openai-responses` require an extension provider).

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [{ "id": "llama3.1:8b" }]
    },
    "plexus": {
      "baseUrl": "https://plexus.home.cowger.us/v1",
      "api": "openai-responses",
      "apiKey": "$AIHOME_API_KEY",
      "authHeader": true,
      "models": [{
        "id": "gpt-5.6-luna",
        "name": "GPT 5.6 Luna",
        "reasoning": true,
        "contextWindow": 400000,
        "maxTokens": 32768,
        "input": ["text", "image"],
        "cost": { "input": 2, "output": 10, "cacheRead": 0.2, "cacheWrite": 2.5 }
      }]
    }
  }
}
```

Key points:
- A provider needs an auth configuration (`apiKey`, even a dummy) before its models appear in `/model`.
- Secret expansion in `apiKey`/`headers`: `$VAR`, `${A}_${B}`, `!command` (e.g. `"!op read op://vault/item/key"`), `$$` → literal `$`, `$!` → literal `!`. `authHeader: true` sends `Authorization: Bearer <key>`.
- Override a built-in provider's endpoint with just `{ "providers": { "anthropic": { "baseUrl": "..." } } }`; `models` merge with built-ins (matching IDs replace); `modelOverrides` patches built-in model metadata (name, compat, cost) without redefining it.
- Model fields: `id`, `name`, `api`, `baseUrl`, `reasoning`, `thinkingLevelMap` (map Pi levels → provider values, `null` hides a level), `input`, `contextWindow` (default 128000), `maxTokens` (default 16384), `cost` (+ `tiers` for context-size pricing steps), `samplingParams`, `headers`, `compat` (OpenAI/Anthropic compatibility knobs like `supportsDeveloperRole`, `supportsReasoningEffort`, `thinkingFormat`).
- Credential resolution order: `--api-key` → `~/.pi/agent/auth.json` → provider env var (e.g. `OPENAI_API_KEY`) → `models.json` `apiKey`. `auth.json` supports per-credential `env` blocks.

Deep dive: [references/models-providers.md](references/models-providers.md)

---

## 3. Keybindings (`keybindings.json`)

Global-only: `~/.pi/agent/keybindings.json`. After editing, run `/reload` (check with `/hotkeys`).

```json
{
  "tui.editor.historyPrevious": "ctrl+n",
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"],
  "app.model.select": "alt+m",
  "app.session.tree": "alt+t"
}
```

- Values: single key string or array; syntax like `ctrl+x`, `shift+enter`, `alt+left`, `super+k`.
- Action IDs are namespaced: `tui.editor.*`, `tui.input.*`, `tui.select.*`, `tui.altScreen.*`, `app.*` (interrupt, clear, exit, suspend), `app.session.*`, `app.model.*`, `app.thinking.*`, `app.tools.*`, `app.message.*`, `app.tree.*`, `app.models.*`.

Full action ID catalog: [references/keybindings.md](references/keybindings.md)

---

## 4. Themes

Built-ins: `dark`, `light`. Custom themes are JSON files in `~/.pi/agent/themes/*.json`, `.pi/themes/*.json`, package `themes/` dirs, or paths listed in `settings.themes`; load ad hoc with `pi --theme ./theme.json`.

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "breezy-ocean",
  "vars": { "primary": "#00aaff" },
  "colors": { "accent": "primary", "text": "", "toolSuccessBg": "#1e2e1e", "thinkingHigh": "#ff00ff", "..." : "..." }
}
```

- Colors: `#rrggbb`, 256-color int (0–255), `vars` name, or `""` (terminal default). `name` required/unique, no `/`.
- The active theme hot-reloads when its file is edited. Select with `"theme": "name"` in settings or one-shot with `pi --use-theme light`.
- Color categories: core UI, message backgrounds, markdown, tool diffs, syntax highlighting, thinking-level borders, bash mode; optional `export` block for HTML export.

---

## 5. Extensions & Packages

Pi has **no native permission popups or sandbox** — approval workflows, custom providers, OAuth, and MCP are all built as extensions.

**Discovery**: `~/.pi/agent/extensions/*.ts` (or `*/index.ts`), `.pi/extensions/`, paths in `settings.extensions`, or `pi -e ./extension.ts` for a one-off load.

**Packages** bundle extensions/skills/prompts/themes. Install with `pi install npm:@foo/bar@1.0.0` / `git:github.com/user/repo@v1` / local path (global: `~/.pi/agent/npm|git/`, project with `-l`: `.pi/npm|git/`). List with `pi list`, update with `pi update`. Resources discovered from package manifest:

```json
{ "name": "my-pi-package", "keywords": ["pi-package"],
  "pi": { "extensions": ["./extensions"], "skills": ["./skills"], "prompts": ["./prompts"], "themes": ["./themes"] } }
```

Or conventionally-named directories when no `pi` block exists. Filter package resources in settings (`!` exclude, `+`/`-` force include/exclude exact path, `[]` none, `autoload: false`):

```json
{ "packages": [ { "source": "npm:pi-mcp-adapter", "skills": ["-skills/mcp-scripting/SKILL.md"] } ] }
```

**Extension API surfaces** (config-relevant): `pi.registerTool()`, `registerCommand()`, `registerShortcut()`, `registerFlag()`, `registerProvider()` / `unregisterProvider()`, `setActiveTools()`, plus events `tool_call` / `tool_result` (build approval gates with `ctx.ui.confirm(...)`), `project_trust`, `before_provider_headers|request`, `session_before_compact`, `resources_discover`, `context`, and custom renderers. Use an extension (not `models.json`) for providers needing OAuth, dynamic model discovery, or nonstandard APIs.

Deep dive: [references/extensions-skills.md](references/extensions-skills.md)

---

## 6. Prompt Templates & Context Files

- **Prompt templates**: `~/.pi/agent/prompts/*.md` and `.pi/prompts/*.md` (non-recursive), package `prompts/`, `settings.prompts` paths, `pi --prompt-template`. Filename becomes the slash command (`review.md` → `/review`). Support optional `description` / `argument-hint` frontmatter and `$1`, `$@`, `$ARGUMENTS`, `${1:-default}`, `${@:N:L}` substitution.
- **Context/system files**: `~/.pi/agent/AGENTS.md` (global rules), `SYSTEM.md`, `APPEND_SYSTEM.md` — global or project (`.pi/`). CLI: `--system-prompt`, `--append-system-prompt`, `--no-context-files`.

Deep dive: [references/extensions-skills.md](references/extensions-skills.md)

---

## 7. Skills

Discovery paths: `~/.pi/agent/skills/`, `~/.agents/skills/` (global); `.pi/skills/`, `.agents/skills/` up the tree (project); package `skills/`; `settings.skills` paths; `pi --skill`.

- Directories containing `SKILL.md` discovered recursively; `.agents/skills/` requires nested dirs (root `.md` ignored); first skill wins on name collision.
- Frontmatter: `name` (1–64 chars, lowercase/digits/hyphens) and `description` (≤1024 chars) required; optional `license`, `compatibility`, `metadata`, `allowed-tools` (space-delimited), `disable-model-invocation`.
- With `enableSkillCommands: true` (default), invoke via `/skill:name [args]` — args append to skill content as user instructions.

---

## 8. MCP via pi-mcp-adapter (nicobailon/pi-mcp-adapter)

> **Core Pi (0.85.x) has no native MCP support.** The community `pi-mcp-adapter` package adds it with a single ~200-token `mcp()` proxy tool instead of burning context on hundreds of tool definitions; servers start lazily on first use.

**Install**: `pi install npm:pi-mcp-adapter`, restart Pi. Pin when reproducibility matters (`npm:pi-mcp-adapter@2.26.1`) — version-specific regressions happen. After any config file change: **`/reload`**.

### Config files & precedence (later wins)

1. `~/.config/mcp/mcp.json` (shared user-global, all MCP hosts)
2. `~/.agents/mcp.json`, 3. `~/.agents/mcp/mcp.json` (tool-agnostic globals)
4. `~/.pi/agent/mcp.json` (Pi global override — adapter-owned server edits land here)
5. `.mcp.json` (project shared)
6. `.pi/mcp.json` (Pi project override — `/mcp disable|enable` writes `disabled` flags here only)

Host configs (Cursor, Claude Code/Desktop, Codex, OpenCode, Windsurf, VS Code) are **not loaded by default** — adopt via `/mcp setup`, `pi-mcp-adapter init`, or `settings.hostConfigDiscovery: "on"`.

### Example (`~/.pi/agent/mcp.json`)

```json
{
  "settings": { "directTools": true },
  "mcpServers": {
    "github": {
      "url": "https://plexus.home.cowger.us/mcp/github",
      "auth": "bearer",
      "bearerTokenEnv": "AIHOME_API_KEY",
      "directTools": ["pull_request_read", "get_file_contents", "search_code"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/workspace"],
      "lifecycle": "lazy"
    },
    "home-assistant": { "url": "...", "auth": "bearer", "bearerTokenEnv": "AIHOME_API_KEY", "disabled": true }
  }
}
```

### Server fields (essentials)

- Transports (mutually exclusive): `command`/`args`/`env`/`cwd` (stdio), `url` (Streamable HTTP, SSE fallback), `socket` (rmcp-mux). Env interpolation `${VAR}`, `~`; secret values may use `!command`.
- Auth: `auth: "oauth" | "bearer" | false`, `bearerToken` / `bearerTokenEnv` / `bearerTokenStore: true` (OS keyring), full `oauth` object (clientId/secret, `redirectUri`, `clientName`, `grantType: "client_credentials"`...). Custom `headers` suppress OAuth auto-detection.
- Behavior: `lifecycle` (`lazy`|`eager`|`keep-alive`|`lazy-keep-alive`), `idleTimeout`, `requestTimeoutMs`, `directTools`, `toolPrefix`, `includeTools`/`excludeTools`, `searchKeywords`, `approveTools`, `exposeResources`, `protocolVersion`, `debug`, `trace`, `disabled`.

### directTools — promote proxy tools to first-class tools

`"directTools": true | ["tool_names"]` globally (`settings`) or per server. Named `<server>_<tool>` (prefix modes `server|short|none|mcp`). Each direct tool costs ~150–300 prompt tokens; keep sets to ~5–20 tools, big catalogs proxy-only. Cache-driven (`~/.pi/agent/mcp-cache.json`) — needs one successful connection before tools appear; force with `/mcp reconnect <server>`. Env override: `MCP_DIRECT_TOOLS=server,server/tool` or `__none__`.

### Commands

| Command | Purpose |
| --- | --- |
| `/mcp` | Server panel: status, per-server/tool direct vs proxy toggles, reconnect (Ctrl+R), enable/disable (Ctrl+D), OAuth (Ctrl+A), save (Ctrl+S) |
| `/mcp status` `tools` `prompts` | Text views |
| `/mcp reconnect [server]` | Reconnect and refresh tool catalog |
| `/mcp setup` | Guided config scaffold/import with diffs |
| `/mcp disable|enable <server>` | Writes `disabled` flag to `.pi/mcp.json` — then `/reload` |
| `/mcp-auth [server]` | OAuth flow (picker or specific server; auto-reconnects) |
| `mcp({search|describe|tool|connect|server|instructions|action})` | The `mcp()` proxy tool itself |

Credentials: OAuth tokens and bearer-store tokens live in the OS keyring (`pi-mcp-adapter.oauth` / `pi-mcp-adapter.bearer` services, keyed by sha256 of server name, URL-bound) — stored plaintext legacy files under `~/.pi/agent/mcp-oauth/` are one-way imported then deleted. Headless Linux without libsecret fails closed (no silent plaintext fallback).

Full adapter guide: [references/pi-mcp-adapter.md](references/pi-mcp-adapter.md)

---

## 9. Project Trust & Tool Control

- Trust gates **config/code loading** (`.pi/` settings, extensions, skills, prompts, themes, SYSTEM.md, packages) — persist with `/trust`, policy `defaultProjectTrust`, per-run `--approve`/`--no-approve`, storage `~/.pi/agent/trust.json` (inheriting from closest trusted parent).
- Tool exposure is controlled by **allowlisting**, not approval: `settings.defaultTools`, `--tools` (strict allowlist), `--exclude-tools`, `--no-builtin-tools`, `--no-tools`. `/skill:name`... extension tools remain controllable via `pi.setActiveTools()`.
- For confirmation flows (e.g. block `rm -rf`), implement a `tool_call` handler extension — see `examples/extensions/permission-gate.ts` in the pi repo. For real isolation, run Pi in a container (see `docs/containerization.md`).

---

## 10. Environment Variables

| Variable | Effect |
| --- | --- |
| `PI_CODING_AGENT_DIR` | Global config dir override (settings, models, auth, keybindings, packages, sessions) |
| `PI_CODING_AGENT_SESSION_DIR` | Session dir override (precedence: `--session-dir` > env > `settings.sessionDir` > default) |
| `PI_PACKAGE_DIR` | Installed package dir override |
| `PI_OFFLINE` | No startup network ops, updates, or telemetry |
| `PI_SKIP_VERSION_CHECK` | Skip latest-version request |
| `PI_TELEMETRY` | `1/0` telemetry override (also `enableInstallTelemetry` setting) |
| `PI_CACHE_RETENTION=long` | Extended provider prompt caching |
| `PI_SHARE_VIEWER_URL` | Base URL for `/share` |
| `PI_HYPERLINKS`, `PI_IMAGE_PROTOCOL`, `PI_TRUE_COLOR` | Terminal capability overrides (`kitty`/`iterm2`/`none`, etc.) |
| `PI_TUI_ESC_TIMEOUT`, `PI_HARDWARE_CURSOR`, `PI_CLEAR_ON_SHRINK` | TUI tuning |
| `VISUAL`, `EDITOR` | External editor fallback (settings `externalEditor` wins) |
| `HTTP_PROXY` / `HTTPS_PROXY` | Proxies (or set `httpProxy` globally) |
| Provider keys | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `OPENROUTER_API_KEY`, ... (see references) |
| Process markers | `AI_AGENT=pi`, `PI_CODING_AGENT=true` exported to children |
| Shell tool session vars | `PI_SESSION_ID`, `PI_SESSION_FILE`, `PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL` |

Adapter-specific: `MCP_DIRECT_TOOLS`, `MCP_OUTPUT_GUARD=0`, `MCP_OAUTH_CALLBACK_PORT` (default 19876), `PI_MCP_ADAPTER_DISABLE_AUTH_CACHE`, `MCP_OAUTH_DIR` (legacy import).

Full list with provider/cloud variables: [references/settings.md](references/settings.md)

---

## 11. Troubleshooting & Diagnostics

| Symptom | Cause | Solution |
| --- | --- | --- |
| Custom model missing from `/model` | Provider/auth not resolved; model has no credentials | `models.json` needs a resolvable `apiKey` (even dummy like `"ollama"`); check `pi --list-models`; `models.json` reloads when `/model` opens |
| Project `.pi/settings.json` ignored | Directory not trusted | `/trust` to persist, `pi --approve`, or `"defaultProjectTrust": "always"` (trust is checked at startup; `/trust` doesn't reload current session) |
| Project array config wiped global values | Arrays replace, never append | List the complete desired array in the highest-precedence file |
| Keybinding doesn't fire | Not reloaded / terminal captures chord / old unnamespaced ID | `/reload`; check effective bindings with `/hotkeys`; use namespaced IDs (`tui.editor.*`, `app.model.*`) |
| `settings.json` rejected | Comments or JSON errors | Settings are strict JSON (unlike some other tools); validate with `node -e 'JSON.parse(...)'` |
| Credential confusion | Resolution order | `--api-key` > `auth.json` > provider env var > `models.json` `apiKey`; `!command` in `models.json` resolves at request time |
| MCP config change not applying | Adapter caches at load | `/reload` after editing any mcp.json; `/mcp reconnect <server>` for live refresh |
| `spawn ENOENT` on stdio MCP server | Bad `cwd` or command | Check `cwd` exists first (misleading error), `command` on PATH, set `"debug": true` for stderr |
| OAuth wants a browser on headless box | No stored creds | `mcp({action:"auth-start", server})` → open URL locally → `auth-complete` with pasted callback URL |
| "Credential storage unavailable" on Linux | No unlocked libsecret/keyring (common in WSL/tmux) | Adapter fails closed; unlock keyring, or use `bearerTokenEnv`/headers instead of OAuth |
| HTTP MCP server connects to wrong thing | URL serves HTML, not MCP | Error says "does not appear to speak MCP" — verify endpoint is the MCP path |
| Direct tools missing after config | Cache not populated | First successful connection populates `mcp-cache.json`; `/mcp reconnect <server>` |
| Extension/skill not loading | Discovery rules | Extensions: `~/.pi/agent/extensions/*.ts` or `*/index.ts`; skills need `SKILL.md` in a dir; check `/reload` and trust |
