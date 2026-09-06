# OpenCode Settings Catalog & Reference

Complete reference for `opencode.json` / `opencode.jsonc` fields (schema: `https://opencode.ai/config.json`). Source of truth: `packages/core/src/v1/config/config.ts` in the opencode repo.

## Top-Level Fields

| Field | Type | Description |
| --- | --- | --- |
| `$schema` | string | JSON schema URL — use `https://opencode.ai/config.json` for editor completion |
| `model` | string | Default model in `provider/model-id` format, e.g. `anthropic/claude-sonnet-4-5` |
| `small_model` | string | Model for lightweight tasks (titles, summaries); falls back to a cheaper model from the provider, then `model` |
| `default_agent` | string | Default primary agent; must be a primary agent; falls back to `build` |
| `subagent_depth` | number | Max subagent nesting depth. Default 1 (subagents cannot spawn subagents) |
| `username` | string | Display name instead of system username |
| `shell` | string | Shell for the interactive terminal and bash tool (auto-detected if unset) |
| `logLevel` | string | `DEBUG` \| `INFO` \| `WARN` \| `ERROR` |
| `share` | string | `manual` \| `auto` \| `disabled` — session sharing behavior |
| `autoupdate` | bool \| "notify" | Auto-update behavior; `notify` shows update notifications only |
| `snapshot` | bool | Filesystem snapshot tracking; when `false`, `/undo` & `/redo` won't revert file changes (default true) |
| `instructions` | string[] | Extra instruction files/patterns (e.g. `~/.agents/APPEND_SYSTEM.md`); **concatenates + dedupes across config layers** (unique among arrays) |
| `disabled_providers` | string[] | Auto-loaded providers to disable |
| `enabled_providers` | string[] | When set, ONLY these providers load — all others ignored |
| `provider` | record | Custom provider configs and model overrides (see providers-models.md) |
| `agent` | record | Agent configs, keyed by name (see agents-commands-skills.md) |
| `mode` | record | **Deprecated** — use `agent`; entries become primary agents |
| `mcp` | record | MCP server configs (see mcp-servers.md) |
| `permission` | object | Permission rules (see permissions.md) |
| `tools` | record<string, bool> | **Deprecated** (v1.1.1+) — merged into `permission` (`true` → allow, `false` → deny; `write`/`edit`/`patch` all map to `edit`) |
| `command` | record | Custom command configs, keyed by name |
| `skills` | object | `{ paths: string[], urls: string[] }` — extra skill folders and remote skill URLs |
| `references` / `reference` | object | Named git or local directory references (`references` preferred) |
| `plugin` | array | Plugin specs: `"pkg"` or `["pkg", {options}]` |
| `formatter` | bool \| object | Enable/configure formatters; omit or `false` to disable, `true` for built-ins |
| `lsp` | bool \| object | Enable/configure LSP servers; same pattern as `formatter` |
| `attachment` | object | Attachment processing (image limits — below) |
| `tool_output` | object | Output truncation thresholds (below) |
| `compaction` | object | Context compaction settings (below) |
| `experimental` | object | Experimental flags (below) |
| `server` | object | Server config for `opencode serve` / `opencode web` (below) |
| `watcher` | object | `{ ignore: string[] }` — file watcher ignore patterns |
| `enterprise` | object | `{ url }` — enterprise URL |
| `layout` | — | **Deprecated** — always stretch layout |

## Compaction

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `compaction.auto` | bool | `true` | Compact automatically when context is full |
| `compaction.prune` | bool | `false` | Prune old tool outputs |
| `compaction.tail_turns` | number | — | Max recent user turns (with their responses) kept verbatim |
| `compaction.preserve_recent_tokens` | number | — | Token budget for verbatim recent-turn retention |
| `compaction.reserved` | number | — | Extra token buffer so compaction itself doesn't overflow the window |

Env overrides: `OPENCODE_DISABLE_AUTOCOMPACT=1` forces `auto: false`; `OPENCODE_DISABLE_PRUNE=1` forces `prune: false`.

Hidden system agents `compaction`, `title`, `summary` run these flows — customize their model/prompt through `agent.<name>`.

## Tool Output Truncation

| Key | Default | Description |
| --- | --- | --- |
| `tool_output.max_lines` | `2000` | Max lines before output is truncated and full text written to the truncation directory |
| `tool_output.max_bytes` | `51200` | Max bytes before truncation |

## Image Attachments

| Key | Default | Description |
| --- | --- | --- |
| `attachment.image.auto_resize` | `true` | Resize images exceeding limits before sending |
| `attachment.image.max_width` | `2000` | Max width in pixels |
| `attachment.image.max_height` | `2000` | Max height in pixels |
| `attachment.image.max_base64_bytes` | `5242880` | Max base64 payload size |

## Experimental Flags

| Key | Description |
| --- | --- |
| `experimental.disable_paste_summary` | Don't summarize large pastes |
| `experimental.batch_tool` | Enable the batch tool |
| `experimental.openTelemetry` | Emit OpenTelemetry spans for AI SDK calls |
| `experimental.primary_tools` | Tools restricted to primary agents |
| `experimental.continue_loop_on_deny` | Continue the agent loop when a tool call is denied |
| `experimental.mcp_timeout` | Timeout (ms) for MCP requests |
| `experimental.policies` | Policy statements, e.g. `[{ "effect": "deny", "action": "provider.use", "resource": "openai" }]` |

## Server Config (`opencode serve` / `opencode web`)

| Key | Description |
| --- | --- |
| `server.port` | Port to listen on |
| `server.hostname` | Hostname; defaults to `0.0.0.0` when mDNS enabled |
| `server.mdns` | Enable mDNS service discovery |
| `server.mdnsDomain` | Custom mDNS domain (default `opencode.local`) |
| `server.cors` | Extra allowed CORS origins — full origins (scheme + host [+ port]), e.g. `https://app.example.com` |

## Config Discovery Details

- **Global**: `~/.config/opencode/` (XDG). Candidate files load in order `config.json` → `opencode.json` → `opencode.jsonc`; a legacy `config` TOML file auto-migrates to `config.json`. If none exist, opencode seeds `opencode.jsonc` with just the `$schema` key.
- **Project**: `opencode.json` / `opencode.jsonc` found by walking from cwd up to the git worktree root (closest file wins).
- **`.opencode` directories**: discovered walking cwd → worktree root, plus `~/.opencode/`, plus `OPENCODE_CONFIG_DIR`. Each contributes `opencode.json(c)`, `tui.json(c)`, agents, commands, and plugins.
- **Remote**: providers authenticated with a `.well-known/opencode` endpoint contribute an org-default layer (lowest precedence).
- **Managed**: `/Library/Application Support/opencode/` (macOS), `/etc/opencode/` (Linux), `%ProgramData%\opencode` (Windows); macOS MDM `.mobileconfig` (domain `ai.opencode.managed`) overrides everything.
- **Post-merge normalization**: `tools` booleans convert to permissions; `mode` entries merge into `agent` as primary; missing `username` falls back to OS username; `autoshare: true` (deprecated) implies `share: "auto"`.
