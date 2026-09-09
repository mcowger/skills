# Codex MCP Servers Reference

Configuration for `[mcp_servers.<name>]` in `~/.codex/config.toml` (project `.codex/config.toml` can add servers for trusted trees). Source of truth: `RawMcpServerConfig` → `McpServerConfig` in `codex-rs/config/src/mcp_types.rs` (unknown fields rejected; new fields must update the mapping). Manage via `codex mcp list|get|add|remove|login|logout`; diagnose via `codex doctor` (MCP section probes reachability; required-server failures fail the report).

## Transport — pick exactly one

```toml
# Streamable HTTP (remote / gateway)
[mcp_servers.github]
url = "https://plexus.home.cowger.us/mcp/github"
bearer_token_env_var = "AIHOME_API_KEY"

# stdio (local subprocess)
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/home/matt.cowger/workspace"]
env = { MY_VAR = "value" }
cwd = "/home/matt.cowger/workspace"
```

| Field | Transport | Meaning |
| --- | --- | --- |
| `url` | HTTP | Streamable-HTTP endpoint. Setting `url` selects HTTP; setting `command` selects stdio — both/neither is an error |
| `command` + `args` | stdio | Executable (absolute or bare name) + argv |
| `env` | stdio | Literal extra env for the child (no expansion — values are literal) |
| `env_vars` | stdio | `["NAME", { name = "NAME", source = "local" \| "remote" }]` — pass-through names with provenance |
| `cwd` | stdio | Working directory for the child |
| `bearer_token_env_var` | HTTP | Env var whose value becomes `Authorization: Bearer <token>`. **The secret lives in the env, never in config** |
| `http_headers` | HTTP | Static extra headers |
| `env_http_headers` | HTTP | `{ "X-Header": "ENV_VAR" }` — per-request env-sourced headers, skipped when unset/empty |
| `http_headers_helper` | HTTP | Local-only shell command printing a JSON object of dynamic headers. Visible in process listings — never emit credentials |
| `bearer_token` | — | Accepted by the parser **only to raise a targeted error** — do not use; use `bearer_token_env_var` |

## Auth fallback

After configured bearer/headers fail to resolve, Codex tries one of:

```toml
[mcp_servers.internal]
url = "https://mcp.corp.example.com"
auth = "chatgpt"     # or "oauth" (default)
scopes = ["read", "write"]
oauth_resource = "https://mcp.corp.example.com"

[mcp_servers.internal.oauth]
client_id = "..."
callback_url = "http://127.0.0.1:8912/callback"
callback_port = 8912
```

| Value | Meaning |
| --- | --- |
| `oauth` (default) | Use stored MCP OAuth credentials; `codex mcp login <name>` starts the flow, `logout` clears it |
| `chatgpt` | Use the current ChatGPT session for first-party-origin servers (falls back to stored OAuth when no session provider exists); unauthenticated last resort |

Global OAuth listener tuning: `mcp_oauth_callback_port`, `mcp_oauth_callback_url` (per-server `oauth.callback_*` wins). Storage backend: `mcp_oauth_credentials_store = "auto" | "keyring" | "file"`.

## Behavior & exposure

```toml
[mcp_servers.flaky]
url = "..."
enabled = false                             # skip at startup (default true)
required = true                             # codex exec exits non-zero if init fails
startup_timeout_sec = 10.0                  # init + initial tool-list budget (ms variant also accepted)
tool_timeout_sec = 60.0                     # per-tool-call default
supports_parallel_tool_calls = true         # advertise all tools as parallel-safe
omit_tools_from = ["deferred"]              # hide from surfaces: code_mode | deferred | direct
default_tools_approval_mode = "prompt"      # auto | prompt | writes | approve
enabled_tools = ["read", "search"]          # allowlist (only these register)
disabled_tools = ["dangerous_write"]        # denylist (applied after the allowlist)
environment_id = "local"                    # where to start the server (default "local")

[mcp_servers.flaky.tools.search]
approval_mode = "prompt"
output_token_limit = 4000
```

- `startup_timeout_sec` accepts fractional seconds; `startup_timeout_ms` is the alternate spelling. `mcp_optional_startup_grace_ms` (top level, default 1000; `0` = wait per-server timeouts) bounds initial catalog building.
- Approval precedence: per-tool `tools.<tool>.approval_mode` > server `default_tools_approval_mode` > global approval policy. `output_token_limit` caps that tool's context cost (+20% serialization allowance).
- Scoping tools to surfaces (`omit_tools_from`) keeps noisy servers out of the initial tool list (`direct`), deferred discovery (`deferred`), or code-mode scripts (`code_mode`).

## Verification

```bash
codex mcp list --json
codex mcp get github
codex doctor            # MCP reachability section
```
