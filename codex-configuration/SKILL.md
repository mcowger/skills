---
name: codex-configuration
description: Configure and customize the Codex harness (openai/codex) settings, model providers, model catalogs, auth, approvals and sandboxing, MCP servers, hooks, skills, TUI keybindings, features, and CLI usage via ~/.codex/config.toml. TRIGGERS - codex config, configure codex, codex settings, config.toml, codex models, model_providers, model catalog, codex login, codex auth, codex sandbox, codex approvals, codex mcp, codex keybindings, codex keymap, codex profile, codex features, codex doctor.
---

# Codex Configuration

A comprehensive guide and reference playbook for configuring, customizing, and troubleshooting **Codex** (openai/codex, Rust harness in `~/workspace/codex`).

## When to Use This Skill

Use this skill whenever you need to:
- Configure Codex settings (`~/.codex/config.toml`, profiles, project-local `.codex/config.toml`)
- Select or define model providers (`openai`, `amazon-bedrock`, `ollama`, `lmstudio`, custom gateways) — including auth, retries, and compatibility options
- Understand or customize the **model catalog** (`/models` endpoint, cache, `model_catalog_json` static catalogs)
- Manage auth (`codex login`, `auth.json`, API keys, ChatGPT login, Bedrock)
- Tune approvals, sandboxing, permissions profiles, and project trust
- Add, update, or troubleshoot MCP servers (stdio, Streamable HTTP, OAuth)
- Configure hooks, skills, memories, plugins, features, or the TUI (theme, status line, keymap)
- Use the Codex CLI (`codex`, `codex exec`, `codex mcp`, `codex doctor`, `-c` overrides, `--profile`)
- Sync Codex configuration across machines using chezmoi

---

## Architecture & Configuration Hierarchy

Config layers load lowest → highest precedence; **TOML tables deep-merge, everything else (scalars, arrays) is replaced** by the higher layer:

| Layer | Location | Notes |
| --- | --- | --- |
| **Packaged defaults** | Embedded `defaults.toml` in the binary | Baseline (`history` save-all, `file_opener` vscode, ...). Overridable via a packaged-defaults path in hosted builds |
| **Admin (MDM)** | macOS managed device profiles | macOS only |
| **System** | `/etc/codex/config.toml` (Unix) or `%ProgramData%\OpenAI\Codex\config.toml` (Windows) | Machine-wide admin config |
| **Cloud** | Enterprise-managed cloud config bundle fragments | Org-managed; only when enrolled |
| **User** | `$CODEX_HOME/config.toml` (`~/.codex/config.toml` by default) | Your persistent personal config — this is the file you edit 99% of the time |
| **Profile** | `$CODEX_HOME/<name>.config.toml` when `--profile <name>` | Layered on top of the base user config; only needs overrides |
| **cwd** | `$PWD/config.toml` | Loaded but **disabled** when the directory is untrusted |
| **Tree** | Parent dirs up to root, `./.codex/config.toml` | Loaded but **disabled** when untrusted |
| **Repo** | `$(git rev-parse --show-toplevel)/.codex/config.toml` | Loaded but **disabled** when untrusted |
| **Runtime** | `-c key=value` flags, TUI model selector | Highest precedence; process-local |
| **Legacy managed** | `/etc/codex/managed_config.toml` (+ MDM) | Back-compat layer applied on top |

Separate **requirements layers** (`requirements.toml`: system → cloud → legacy → admin) *constrain* what config may select (allowed sandbox modes, approval policies, login methods, MCP/plugin rules) — they narrow choices, they don't set values.

**Project trust:** project-local layers (cwd/tree/repo) only apply when the directory tree is trusted via `[projects."/path"] trust_level = "trusted"`. Untrusted trees fall back to restrictive behavior. Project-local config **cannot** set credential-routing keys (`openai_base_url`, `chatgpt_base_url`, `model_provider`, `model_providers`, `notify`, `profile`/`profiles`, `otel`, realtime overrides) — those are silently ignored from project layers.

### Key Files & Directories

| Path | Format | Purpose |
| --- | --- | --- |
| `~/.codex/config.toml` (`$CODEX_HOME`) | TOML | Main settings: model, provider, approvals, MCP, TUI, features |
| `~/.codex/<name>.config.toml` | TOML | Named profile (`codex --profile <name>`) |
| `~/.codex/auth.json` | JSON | CLI auth credentials (or OS keyring — see `cli_auth_credentials_store`) |
| `~/.codex/models_cache.json` | JSON | Cached `/models` catalog (TTL 5 min, identity-scoped) |
| `~/.codex/history.jsonl` | JSONL | Prompt history (`[history]` controls persistence) |
| `~/.codex/skills/` | `SKILL.md` dirs | Bundled/locally-installed skills |
| `~/.codex/hooks.json` | JSON | Hook definitions (trust hashes in config `[hooks.state]`) |
| `~/.codex/log/` | logs | Log dir (override with `log_dir`) |
| `<repo>/.codex/config.toml` | TOML | Project-scoped overrides (needs trust) |
| `$PWD/config.toml` | TOML | cwd-scoped config (needs trust) |

`CODEX_HOME` env var overrides `~/.codex` — when set, the directory **must already exist**. Use `--strict-config` (or `-c`) runs to fail fast on unknown fields (typo detection).

---

## Quick Reference CLI Commands

```bash
codex                                   # Start the TUI in the current project
codex "do the thing"                    # TUI with an initial prompt
codex exec "do the thing"               # Non-interactive run (alias: codex e)
codex exec --json "..."                 # JSONL event stream on stdout
codex exec -o out.txt "..."             # Write last message to a file
codex exec --output-schema schema.json  # Constrain final response shape
codex review                            # Non-interactive code review

# Auth
codex login                             # ChatGPT login (browser)
codex login --with-api-key < <(printenv OPENAI_API_KEY)
codex login --with-access-token < <(printenv CODEX_ACCESS_TOKEN)
codex login --device-auth               # Device-code flow (headless)
codex login status                      # Show login status
codex logout                            # Remove stored credentials

# Models / providers / MCP / misc
codex debug models                      # Render the raw active model catalog as JSON
codex mcp list [--json]                 # List configured MCP servers
codex mcp get <name> [--json]           # Show one server's config
codex mcp add <name> --url <url>        # Add a Streamable-HTTP server
codex mcp add <name> -- <cmd> [...]     # Add a stdio server
codex mcp remove <name>                 # Delete a server entry
codex mcp login <name>                  # OAuth login for an MCP server
codex mcp logout <name>                 # Remove MCP OAuth credentials
codex doctor [--json] [--summary]       # Diagnose install, config, auth, MCP, runtime
codex resume [--last]                   # Resume a previous session
codex fork [--last]                     # Fork a previous session
codex queue / archive / delete / unarchive
codex features                          # Inspect feature flags
codex completion bash                   # Shell completions
codex update                            # Update Codex

# Shared run flags (TUI + exec)
codex -m gpt-5.6-luna                   # Model for this run
codex -p work                           # Layer ~/.codex/work.config.toml
codex -c model=gpt-5.6-luna             # Ad-hoc TOML override (repeatable)
codex -c 'sandbox_mode="workspace-write"'
codex -s workspace-write                # Sandbox for this run
codex -a on-request                     # Approval policy for this run
codex --oss [--local-provider ollama]   # Route to a local OSS provider
codex -C ~/repo --add-dir ~/other       # Working dir + extra writable dirs
codex --worktree                        # Run in a new managed git worktree
codex --approve-for-me                  # Auto-review approvals (workspace-write sandbox)
codex --dangerously-bypass-approvals-and-sandbox  # yolo; externally-sandboxed envs only
```

`-c key=value` parses the value as TOML (falls back to raw string): `-c model=o3` works unquoted. Root-level `-c` flags have *lower* precedence than subcommand-level flags.

---

## 1. Main Settings (`config.toml`)

TOML at `~/.codex/config.toml`. The fields you'll touch most — full catalog in [references/settings.md](references/settings.md):

```toml
model = "gpt-5.6-luna"
model_provider = "custom"
model_reasoning_effort = "max"
model_reasoning_summary = "auto"     # auto | concise | detailed | none
model_verbosity = "high"             # low | medium | high (GPT-5 text.verbosity)
service_tier = "priority"            # default | priority | flex (legacy fast works)
plan_mode_reasoning_effort = "high"
review_model = "gpt-5.6-terra"
web_search = "disabled"              # disabled | cached (default) | indexed | live

approval_policy = "on-request"       # untrusted | on-request | granular | never
sandbox_mode = "workspace-write"     # read-only (default) | workspace-write | danger-full-access

[tui]
theme = "dark"

[history]
persistence = "save-all"             # save-all | none

[projects."/home/matt.cowger"]
trust_level = "trusted"              # trusted | untrusted
```

Notables:
- Legacy `[profiles.*]` tables + `profile = "..."` selector still parse but **conflict with `--profile`** — if the base user config contains a legacy `profile = "<name>"` selector or `[profiles.<name>]` table for the requested name, `--profile <name>` errors out. Migrate legacy profiles to `$CODEX_HOME/<name>.config.toml` files.
- `model` is a bare slug (`gpt-5.6-luna`), **not** `provider/model` — the provider comes from `model_provider`.
- `model_catalog_json` pins a static catalog file (startup-only; see section 2).
- Unknown top-level fields are ignored unless `--strict-config` is passed — use it to catch typos.

---

## 2. Providers, Model Catalog & Compatibility

> **Mental model:** Codex talks to exactly one *model provider* (a Responses-API-compatible HTTP endpoint). The *model catalog* describes which model slugs exist and what each one supports. The catalog normally comes from the provider's live `/models` endpoint; `model_catalog_json` replaces it with a static file.

Deep dive: [references/providers-models.md](references/providers-models.md)

### 2a. Selecting a provider

```toml
model_provider = "custom"   # key into [model_providers.*]; default "openai"
openai_base_url = "https://proxy.example.com/v1"  # override ONLY the built-in openai endpoint
oss_provider = "ollama"     # preferred local provider: ollama | lmstudio (for --oss)
```

Built-in providers (compiled in — do not redefine these IDs):

| ID | Auth | Notes |
| --- | --- | --- |
| `openai` | `requires_openai_auth` (ChatGPT/API key via `codex login`) | WebSockets + standalone web search; `base_url` overridable via `openai_base_url` |
| `amazon-bedrock` / `amazon-bedrock-runtime` | AWS (`aws` block) | Only providers that accept the `aws` block; static model catalogs (`openai.gpt-*` slugs; runtime adds `global.`/`us.` routing variants) |
| `ollama` (port 11434) / `lmstudio` (port 1234) | none | Local OSS; `CODEX_OSS_PORT` / `CODEX_OSS_BASE_URL` override the endpoint experimentally |

### 2b. Custom providers (gateways, proxies, third parties)

```toml
[model_providers.custom]
name = "OpenAI via Plexus"
base_url = "https://plexus.home.cowger.us/v1"
wire_api = "responses"          # the ONLY accepted value (`chat` was removed)
env_key = "AIHOME_API_KEY"      # env var holding the bearer token
requires_openai_auth = false    # true = force codex-login auth flow
supports_websockets = false
supports_standalone_web_search = false
```

Auth is **exactly one** of `env_key` (+ optional `env_key_instructions`), `auth.command` (shells out for a bearer token), `experimental_bearer_token` (discouraged — secrets in config), or `requires_openai_auth = true`. AWS `aws = { profile, region, credential_export, auth_refresh }` is Bedrock-only. Extras: `http_headers` / `env_http_headers` (env-sourced headers, skipped when empty), `query_params`, `request_max_retries` (default 4, cap 100), `stream_max_retries` (default 5, cap 100), `stream_idle_timeout_ms` (default 300000), `websocket_connect_timeout_ms` (default 15000). **Never put secrets in `config.toml`** — use `env_key` so they stay in the environment.

### 2c. How the catalog works

1. **Bundled fallback** compiled into the binary (always available offline).
2. **Live refresh**: `GET {base_url}/models` (5 s timeout) with ETag revalidation; cached to `~/.codex/models_cache.json` (TTL 300 s, scoped by client version + a sha256 of provider/auth identity — ChatGPT tokens excluded so rotation doesn't bust the cache).
3. **Static override**: `model_catalog_json = "/path/to/catalog.json"` (`{"models": [...]}`) makes the catalog authoritative and in-memory — no network, no cache writes. Applied **on startup only**; per-thread overrides are no-ops.

Slug resolution is longest-prefix match plus a one-segment namespace fallback (`custom/gpt-5.3-codex` → `gpt-5.3-codex`); unknown slugs get fallback metadata (272k context) with a warning. Per-model **capability/compat fields** in each catalog entry control what Codex attempts: `shell_type` (`shell_command` vs `unified_exec`), `tool_mode` (`code_mode_only`), `use_responses_lite`, `supports_parallel_tool_calls`, `supports_image_detail_original`, `input_modalities`, `truncation_policy`, `web_search_tool_type`, `apply_patch_tool_type`, `supported_reasoning_levels` + `default_reasoning_level`, `context_window` / `max_context_window` / `auto_compact_token_limit`, `service_tiers` / `additional_speed_tiers`, `multi_agent_version`. (There is no provider-side model allowlist — visibility comes from the catalog's `visibility` field and auth-mode filtering.) Verify with `codex debug models`.

Config-side clamps: `model_context_window` (capped at the model's max), `model_auto_compact_token_limit` (+ `_scope`), `tool_output_token_limit`, `personality` (`none` strips the personality section; `friendly`/`pragmatic`).

This machine's pattern (chemoi-tracked): a custom gateway provider + a proxied static catalog snapshot. Refresh the snapshot when the gateway adds models.

---

## 3. Auth

```bash
codex login              # ChatGPT OAuth (browser)
codex login --device-auth
printenv OPENAI_API_KEY | codex login --with-api-key
codex login status
codex logout
```

Auth modes: `ApiKey`, `Chatgpt` (Codex-managed OAuth), `ChatgptAuthTokens` / `Headers` (externally-supplied), `AgentIdentity`, `PersonalAccessToken`, `BedrockApiKey`, `BedrockAccessKeys`. Resolution: `OPENAI_API_KEY` → `CODEX_API_KEY` → `CODEX_ACCESS_TOKEN` env vars, else `codex login` credentials in `$CODEX_HOME/auth.json`. Credential storage: `cli_auth_credentials_store = "file"` (default) | `keyring` | `auto` | `ephemeral`; MCP OAuth separately via `mcp_oauth_credentials_store = "auto"` (default) | `keyring` | `file`. Lock down login with `forced_login_method` and `forced_chatgpt_workspace_id` (string or list — comma-joined strings are rejected). Never commit `auth.json`.

---

## 4. Approvals, Sandbox, Permissions & Trust

```toml
approval_policy = "on-request"
approvals_reviewer = "auto_review"   # with --approve-for-me behavior
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["/scratch"]
network_access = false
```

- `approval_policy`: `untrusted` (approve unless exec-policy allows — used for untrusted projects) | `on-request` (default; model decides; `on-failure` is an alias) | `granular {...}` (per-flow allow/reject without prompting) | `never` (failures go straight to the model).
- `sandbox_mode`: `read-only` (default) | `workspace-write` (+ `[sandbox_workspace_write]` tuning) | `danger-full-access`.
- Named permission profiles: `default_permissions = "my-profile"` (or `":built-in"`), `[permissions.my-profile]` with `extends` inheritance.
- Project trust: `[projects."/abs/path"] trust_level = "trusted"`. Trust lookup checks cwd, then project root (markers in `project_root_markers`, default `[".git"]`), then git repo root. Untrusted → project layers disabled + restrictive approvals.
- Exec-policy rules: `codex execpolicy check` validates `.rules` files; `--ignore-rules` skips user/project rules.

Deep dive: [references/approvals-sandbox.md](references/approvals-sandbox.md)

---

## 5. MCP Servers

```toml
[mcp_servers.github]
url = "https://plexus.home.cowger.us/mcp/github"
bearer_token_env_var = "AIHOME_API_KEY"

[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/home/matt.cowger/workspace"]
startup_timeout_sec = 10.0

[mcp_servers.flaky]
url = "..."
enabled = false          # skip at startup
required = true          # codex exec FAILS if this server won't initialize
enabled_tools = ["read", "search"]   # allowlist; disabled_tools is the denylist
default_tools_approval_mode = "prompt"  # auto | prompt | writes | approve

[mcp_servers.flaky.tools.search]
approval_mode = "prompt"
output_token_limit = 4000
```

- Transports are exclusive: `url` (Streamable HTTP) **or** `command` (+ `args`, `env`, `env_vars`, `cwd`). Never commit secrets — `bearer_token_env_var`, `env_http_headers`, or `http_headers_helper` (prints JSON headers; keep credentials out of process listings).
- Auth fallback after configured credentials: `auth = "oauth"` (default; uses stored OAuth — `codex mcp login`) or `"chatgpt"` (first-party origin via ChatGPT session). Tune OAuth with `scopes`, `oauth = { client_id, callback_url, callback_port }`, `oauth_resource`, global `mcp_oauth_callback_port` / `mcp_oauth_callback_url`.
- `mcp_optional_startup_grace_ms` (default 1000; `0` waits per-server `startup_timeout_sec` instead).

Deep dive: [references/mcp-servers.md](references/mcp-servers.md)

---

## 6. Hooks, Skills, Memories, Plugins & Features

**Hooks** (`~/.codex/hooks.json` + `[hooks]` in config) — events: `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PreCompact`, `PostCompact`, `SessionStart/End`, `UserPromptSubmit`, `SubagentStart/Stop`, `Stop`, `Interrupt`. Trust is hash-pinned in `[hooks.state]`; new/changed hooks prompt before running. `--dangerously-bypass-hook-trust` is for vetted automation only.

```toml
[skills]
include_instructions = true
max_context_tokens = 4000
[[skills.config]]  # enablement rules; later rules win
name = "pdf"
enabled = false
```

**Skills** live in `~/.codex/skills/<name>/SKILL.md` (+ project/plugin roots); `[skills]` tunes the catalog block (`bundled`, `include_instructions`, `max_context_tokens`) and per-skill `[[skills.config]]` `{ path | name, enabled }` rules.

**Memories** (`[memories]`): `version`, `dual_write`, `generate_memories`, `use_memories`, `dedicated_tools`, `disable_on_external_context` (alias `no_memories_if_mcp_or_web_search`), `extract_model` / `consolidation_model` (default `gpt-5.6-luna` / `gpt-5.6-terra`), retention knobs (`max_raw_memories_for_consolidation`, `max_unused_days`, `max_rollout_age_days`, `max_rollouts_per_startup`, `min_rollout_idle_hours`, `min_rate_limit_remaining_percent`).

**Plugins/marketplaces**: `[plugins.<name>]`, `[marketplaces.<name>]` (+ `codex plugin` subcommands). **Agents**: `[agents]` (`enabled`, `max_concurrent_threads_per_session` — alias `max_threads` — `max_depth`, `default_subagent_model` + `_reasoning_effort`). **Features** (`[features.<key>] = true/false`, `--enable/--disable` flags, `/experimental` in TUI): stable `multi_agent`, `apps`, `hooks`, `shell_tool`, `unified_exec`; opt-in stable `multi_agent_v2`; experimental `network_proxy`, `worktrees`; dev-stage keys exist but stay off. **Prompt shaping**: `instructions`, `developer_instructions`, `model_instructions_file` (discouraged — degrades the tuned prompt), `include_{permissions,apps,collaboration_mode,environment_context}_instructions`, `personality`, `compact_prompt` / `experimental_compact_prompt_file`, `notify = ["cmd", ...]`, `web_search` + `[tools.web_search]`.

---

## 7. TUI, Themes & Keybindings

```toml
[tui]
theme = "dark"                 # /theme to switch; custom themes under $CODEX_HOME/themes
status_line = ["model-with-reasoning", "current-dir", "thread-name"]
terminal_title = ["activity", "thread-name", "project-name"]
alternate_screen = "auto"      # auto | always | never (also --no-alt-screen)
vim_mode_default = false
session_picker_view = "dense"  # dense | comfortable
resume_cwd = "session"         # current | session (else prompt when they differ)

[tui.keymap.chat]
previous_permission_mode = "ctrl-p"   # NOTE: these two actions are stripped from project layers
```

Contexts: `global`, `chat`, `composer`, `editor`, `vim_normal`, `vim_operator`, `vim_search`, `vim_text_object`, `pager`, `list`, `agents`, `approval`. Binding = one key or two-stroke chord (`"ctrl-x ctrl-s"`), a list = alternatives, `[]` = unbind. Canonicalize with `ctrl/alt/shift/super` + key (`esc`, `enter`, `tab`, `page-up`, `f1`–`f24`); unknown fields rejected. Also: `file_opener = "vscode"` (URI-scheme opener for citations), `hide_agent_reasoning` / `show_raw_agent_reasoning`, `[history] max_bytes`, `disable_paste_burst`.

Full action catalog: [references/tui-keybindings.md](references/tui-keybindings.md)

---

## 8. Profiles (`--profile`)

```bash
codex --profile work        # layers ~/.codex/work.config.toml over config.toml
```

A profile file contains **only overrides** (same schema as `config.toml`). Only a curated subset is profile-scoped (model, provider, approvals, sandbox, reasoning, verbosity, service tier, personality, instructions, tools/web-search, tui subset, windows, features, oss_provider, chatgpt_base_url, model_catalog_json, analytics). Do **not** combine `--profile <name>` with legacy `profile = "<name>"` / `[profiles.<name>]` in the base config — that errors; migrate to `<name>.config.toml` files.

---

## 9. Variable/Path Conventions

- Paths in `config.toml` resolve **relative to the config file that defines them** (`~/` supported) — keep gateway/proxy file references (like `model_catalog_json`) absolute so profiles and project layers can't shift them.
- MCP `env` values are literal strings (no expansion); reach for `env_vars` (local/remote-sourced names), `env_http_headers`, or `bearer_token_env_var` instead of baking secrets into config.
- Provider `env_key` reads the process environment at request time; `env_key_instructions` tells the user how to obtain the value when it's missing.
- `-c` overrides take dotted paths (`-c features.multi_agent_v2=true`, `-c 'mcp_servers.github.enabled=false'`).

---

## 10. Chezmoi Dotfile Sync & Persistence

Codex files are versioned via chezmoi (`~/workspace/dotfiles`):

| Home file | Chezmoi source |
| --- | --- |
| `~/.codex/config.toml` | `private_dot_codex/private_config.toml` |
| `~/.codex/model-catalog-proxy.json` | `private_dot_codex/model-catalog-proxy.json` |

Do **not** chezmoi-add `auth.json` (secrets), `models_cache.json`, `history.jsonl`, `*.sqlite*`, `sessions/`, `tmp/`, or `shell_snapshots/` — runtime state. `hooks.json` may be tracked, but its `[hooks.state]` trust hashes live in `config.toml` and must travel with it.

```bash
chezmoi diff ~/.codex
chezmoi add ~/.codex/config.toml
chezmoi apply
```

Secrets rule: `config.toml` must contain zero secret values — provider credentials via `env_key`, MCP via `bearer_token_env_var`/`env_http_headers`. The static catalog snapshot (`model-catalog-proxy.json`) is a point-in-time copy — refresh it when the gateway's model list changes.

---

## 11. Troubleshooting & Diagnostics

| Symptom | Cause | Solution |
| --- | --- | --- |
| `Built-in providers cannot be overridden` | `[model_providers.openai]` (or `ollama`/`lmstudio`) redefined | Rename to a custom ID (e.g. `openai-custom`); only Bedrock pair accept partial overrides (`base_url`, `auth`, `http_headers`, `aws.profile/region/credential_export/auth_refresh`) |
| `` `wire_api = "chat"` rejected `` | Chat wire API removed | Set `wire_api = "responses"` |
| `` `ollama-chat` / `--local-provider ollama-chat` rejected `` | Legacy provider ID removed | Use `ollama` |
| Provider auth error at startup | `env_key` var empty/missing, or conflicting auth fields | `auth` can't combine with `env_key`/`experimental_bearer_token`/`requires_openai_auth`; `aws` can't combine with any of those or `supports_websockets`; check `codex login status`, `codex doctor` |
| Model missing from picker | Not in active catalog, or backend-only model without backend auth | `codex debug models` (raw catalog); check `model_catalog_json` freshness; backend-only models need ChatGPT/API auth, not bare `env_key` |
| Custom gateway models behave oddly | No catalog entry → fallback metadata | Add entries to the static catalog (shell_type, tool_mode, reasoning levels, context window); or confirm the gateway's `/models` returns them |
| Config change not taking effect | Layer precedence / untrusted project / per-thread no-op | `model_catalog_json` is startup-only; project layers need `[projects]` trust; `-c` beats files; `--strict-config` catches typos in unknown fields |
| `--profile` errors about legacy profile | Base config still has `profile = "<name>"` / `[profiles.<name>]` | Move settings to `~/.codex/<name>.config.toml`, remove legacy keys |
| MCP server won't connect | Bad URL/command, missing env, timeout, OAuth | `codex mcp list/get`; raise `startup_timeout_sec`; `codex mcp login <name>`; `codex doctor` MCP section; required servers fail `codex exec` by design |
| Hook keeps prompting / won't run | Trust hash mismatch after editing hooks | Re-approve the hook so `[hooks.state]` records the new hash |
| Slash command for schema | Editing codex itself, not configuring it | After changing `ConfigToml` in source, run `just write-config-schema` in `~/workspace/codex` |
| Need a full health picture | — | `codex doctor` (add `--json` for machine-readable, `--summary` for compact) |
