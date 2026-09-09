# Codex Settings Catalog & Reference

Complete reference for `~/.codex/config.toml` top-level fields. Source of truth: `ConfigToml` in `codex-rs/config/src/config_toml.rs` (JSON Schema emitted to `codex-rs/core/config.schema.json` — regenerate with `just write-config-schema` after changing config types). TOML tables deep-merge across layers; scalars and arrays replace.

## Model selection & behavior

| Key | Type | Description |
| --- | --- | --- |
| `model` | string | Model slug for new turns (bare slug, e.g. `"gpt-5.6-luna"` — **not** `provider/model`). Longest-prefix match against the catalog; unknown slugs get fallback metadata + warning |
| `review_model` | string | Model override for the `/review` feature |
| `model_provider` | string | Key into `[model_providers]` (default `"openai"`) |
| `model_context_window` | int | Override the model's context window (clamped to the catalog `max_context_window`) |
| `model_auto_compact_token_limit` | int | Token usage threshold that triggers auto-compaction |
| `model_auto_compact_token_limit_scope` | string | Whether the limit applies to the full context or only tokens after the carried prefix |
| `model_reasoning_effort` | string | `none` \| `minimal` \| `low` \| `medium` (default) \| `high` \| `xhigh` \| `max` \| `ultra` \| `persistent` (unknown future values pass through as custom) |
| `plan_mode_reasoning_effort` | string | Reasoning effort used in plan mode |
| `model_reasoning_summary` | string | `auto` (default) \| `concise` \| `detailed` \| `none` |
| `model_verbosity` | string | GPT-5 Responses `text.verbosity`: `low` \| `medium` (default) \| `high` |
| `service_tier` | string | Explicit service tier id, e.g. `default`, `priority`, `flex` (legacy `fast` works) |
| `personality` | string | `none` (strips personality section) \| `friendly` \| `pragmatic` |
| `model_instructions_file` | path | File whose contents **replace** the built-in model instructions. Strongly discouraged — deviating from sanctioned instructions degrades performance |
| `model_catalog_json` | path | Static `{"models": [...]}` catalog, applied on startup only (see providers-models.md) |
| `review_model` | string | (see above) |
| `tool_output_token_limit` | int | Token budget for stored tool/function outputs (drives per-model truncation policy) |
| `compact_prompt` | string | Custom history-compaction prompt |
| `experimental_compact_prompt_file` | path | File supplying the compaction prompt |
| `experimental_use_unified_exec_tool` | bool | Prefer the unified exec tool |

## Providers & auth

| Key | Type | Description |
| --- | --- | --- |
| `model_providers` | table | Custom providers, `[model_providers.<id>]` (see providers-models.md). Built-in IDs (`openai`, `ollama`, `lmstudio`) cannot be redefined |
| `openai_base_url` | string | Override **only** the built-in `openai` provider's base URL |
| `chatgpt_base_url` | string | Base URL for ChatGPT (non-API) requests |
| `oss_provider` | string | Preferred local provider: `ollama` \| `lmstudio` (the legacy `ollama-chat` id is rejected with a fix-up error) |
| `forced_login_method` | string | Restrict login: `chatgpt` \| `api` |
| `forced_chatgpt_workspace_id` | string \| list | Restrict ChatGPT login to workspace ID(s). Comma-joined strings rejected — use a TOML list |
| `cli_auth_credentials_store` | string | `file` (default) \| `keyring` \| `auto` (keyring else file) \| `ephemeral` (memory only) |
| `mcp_oauth_credentials_store` | string | `auto` (default) \| `keyring` \| `file` |
| `mcp_oauth_callback_port` | int | Fixed port for the MCP OAuth local callback server (else ephemeral) |
| `mcp_oauth_callback_url` | string | Redirect URI sent in the OAuth request (listener still binds 127.0.0.1) |

## Approvals, sandbox, permissions

| Key | Type | Description |
| --- | --- | --- |
| `approval_policy` | string | `untrusted` \| `on-request` (default; `on-failure` alias) \| `granular` \| `never` |
| `approvals_reviewer` | string | Who reviews escalated approvals: `user` (default) \| `auto_review` |
| `auto_review` | table | `{ policy = "..." }` — extra guardian prompt instructions |
| `sandbox_mode` | string | `read-only` (default) \| `workspace-write` \| `danger-full-access` |
| `sandbox_workspace_write` | table | `{ writable_roots, network_access, exclude_tmpdir_env_var, exclude_slash_tmp }` |
| `default_permissions` | string | Permission profile name (`:built-in` for built-ins, else `[permissions]` table) |
| `permissions` | table | Named `[permissions.<name>]` profiles with `extends` inheritance |
| `allow_login_shell` | bool | Whether the model may request a login shell (default true; false rejects `login = true` and defaults omission to non-login) |
| `allow_symlinked_codex_home` | bool | macOS only: allow sandbox writable roots under CODEX_HOME to traverse symlinks. Read from host user config at startup; default false |
| `projects` | table | `[projects."/abs/path"] trust_level = "trusted" \| "untrusted"` |
| `project_root_markers` | list | Markers for project-root detection (default `[".git"]`) |

## MCP

| Key | Type | Description |
| --- | --- | --- |
| `mcp_servers` | table | `[mcp_servers.<name>]` server configs (see mcp-servers.md) |
| `mcp_optional_startup_grace_ms` | int | Grace while building the initial tool catalog (default 1000; `0` waits per-server `startup_timeout_sec`) |

## Prompt shaping & context

| Key | Type | Description |
| --- | --- | --- |
| `instructions` | string | System instructions |
| `developer_instructions` | string | Developer-role message |
| `include_permissions_instructions` | bool | Inject `<permissions instructions>` block (default true) |
| `include_apps_instructions` | bool | Inject `<apps_instructions>` block (default true) |
| `include_collaboration_mode_instructions` | bool | Inject `<collaboration_mode>` block (default true) |
| `include_environment_context` | bool | Inject `<environment_context>` block (default true) |
| `project_doc_max_bytes` | int | Max project-doc (AGENTS.md et al) bytes across environments (default 32768) |
| `project_doc_fallback_filenames` | list | Fallback filenames when AGENTS.md is missing (default `[]`) |
| `notify` | list | External command argv for end-user notifications |
| `web_search` | string | `disabled` \| `cached` (default) \| `indexed` \| `live` |
| `tools` | table | `[tools.web_search]`, `experimental_request_user_input`, `update_plan` toggles |

## Agents, memories, skills, hooks, plugins

| Key | Type | Description |
| --- | --- | --- |
| `agents` | table | `enabled`, `max_concurrent_threads_per_session` (alias `max_threads`), `max_depth` (V1), `default_subagent_model`, `default_subagent_reasoning_effort`, `interrupt_message`, plus `[agents.<role>]` role declarations (`description`, `config_file`, `nickname_candidates`) |
| `memories` | table | `version`, `dual_write`, `generate_memories`, `use_memories`, `dedicated_tools`, `disable_on_external_context` (alias `no_memories_if_mcp_or_web_search`), `extract_model`, `consolidation_model`, retention knobs (see SKILL.md §6) |
| `goals` | table | `max_goal_token_budget` |
| `skills` | table | `bundled`, `include_instructions`, `max_context_tokens`, `[[skills.config]]` enablement rules |
| `hooks` | table | Lifecycle hooks by event + `[hooks.state]` trust hashes |
| `plugins` | table | `[plugins.<name>]` user-level plugin entries |
| `marketplaces` | table | `[marketplaces.<name>]` marketplace entries |
| `tool_suggest` | table | Additional discoverable tool suggestions |
| `orchestrator` | table | Orchestrator-owned `{ skills.enabled, mcp.enabled }` |

## Profiles

| Key | Type | Description |
| --- | --- | --- |
| `profile` | string | **Legacy** selector — conflicts with `--profile <same-name>`; migrate to `<name>.config.toml` |
| `profiles` | table | **Legacy** `[profiles.<name>]` tables — same conflict rule |

`--profile <name>` layers `$CODEX_HOME/<name>.config.toml` over the base config. Profile-scoped subset: model, service_tier, model_provider, approval_policy, approvals_reviewer, sandbox_mode, reasoning efforts/summary/verbosity, model_catalog_json, personality, chatgpt_base_url, model_instructions_file, compact prompt file, unified-exec flag, instruction-inclusion flags, tools, web_search, analytics, `[tui]` subset, `[windows]`, `[features]`, `oss_provider`.

## TUI & history

| Key | Type | Description |
| --- | --- | --- |
| `tui` | table | Notifications, animations, whimsy, tooltips, vim mode, alt-screen, status line, terminal title, theme, pet, session picker, resume cwd, keymap (see tui-keybindings.md) |
| `file_opener` | string | URI-scheme opener for file citations (`vscode`, ...) |
| `hide_agent_reasoning` | bool | Hide `AgentReasoning` events (default false) |
| `show_raw_agent_reasoning` | bool | Show raw reasoning events (default false) |
| `disable_paste_burst` | bool | Legacy alias for `tui.disable_paste_burst` |
| `history` | table | `{ persistence = "save-all" \| "none", max_bytes }` |
| `sqlite_home` | path | State DB dir (`$CODEX_SQLITE_HOME`, else `$CODEX_HOME`) |
| `log_dir` | path | Log dir (also enables TUI text log; default `$CODEX_HOME/log`) |
| `thread_unload_delay_secs` | int | Idle seconds before the app-server unloads a thread (default 60; `0` = immediate; restart required) |
| `background_terminal_max_timeout` | int | Max poll window for background terminal output in ms (default 300000) |

## Misc

| Key | Type | Description |
| --- | --- | --- |
| `shell_environment_policy` | table | Spawned-process env policy: `inherit`, `ignore_default_excludes`, legacy `exclude`/`include_only`/`set`, canonical `filters`, `experimental_use_profile` |
| `features` | table | `[features.<key>]` flags (unknown keys rejected; see SKILL.md §6) |
| `suppress_unstable_features_warning` | bool | Silence under-development feature warnings |
| `analytics` / `feedback` | table | `{ enabled }` (false disables) |
| `check_for_update_on_startup` | bool | Update check (default true; false only if centrally managed) |
| `notice` | table | Acknowledged in-product notices (migration prompts, warnings) |
| `apps` / `apps_mcp_product_sku` | table/string | App-specific controls; SKU forwarded on host-owned Apps MCP requests |
| `responses_api_metadata` | map | Bounded product-owned metadata on every Responses request |
| `otel` | table | OpenTelemetry exporter config |
| `windows` | table | Windows-specific config |
| `audio` / `realtime` | tables | Machine-local realtime audio prefs; realtime session selection (experimental) |
| `browser_use` / `computer_use` | tables | Browser/computer-use config |
| `desktop` | map | Opaque desktop settings |
| `ghost_snapshot` | table | Compatibility-only no-ops (legacy `ghost_snapshot` config still loads) |
| `experimental_thread_store` | table | `local` (or removed `experimental_thread_store_endpoint`, which now fails fast) |
| `experimental_realtime_*` | various | **Do not use** — realtime transport overrides |
