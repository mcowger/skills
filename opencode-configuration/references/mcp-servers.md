# OpenCode MCP Servers Reference

Source of truth: `packages/core/src/v1/config/mcp.ts`.

Servers are defined under `mcp`, keyed by unique name. MCP tools consume model context — enable only the servers you need.

## Local servers (stdio)

```jsonc
{
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"],
      "environment": { "MY_ENV_VAR": "my_env_var_value" },
      "cwd": "./subdir",
      "enabled": true,
      "timeout": 5000
    }
  }
}
```

| Field | Required | Description |
| --- | --- | --- |
| `type` | yes | Must be `"local"` |
| `command` | yes | Executable + arguments as an array |
| `environment` | no | Env vars for the server process |
| `cwd` | no | Working directory; relative paths resolve from the workspace |
| `enabled` | no | Enable/disable the server on startup |
| `timeout` | no | Timeout in ms for MCP requests (default 5000) |

## Remote servers (Streamable HTTP)

```jsonc
{
  "mcp": {
    "exa": {
      "type": "remote",
      "url": "https://plexus.home.cowger.us/mcp/exa",
      "headers": { "Authorization": "Bearer {env:AIHOME_API_KEY}" },
      "enabled": true,
      "timeout": 5000
    },
    "github": {
      "type": "remote",
      "url": "https://plexus.home.cowger.us/mcp/github",
      "oauth": false,
      "headers": { "Authorization": "Bearer {env:AIHOME_API_KEY}" }
    }
  }
}
```

| Field | Required | Description |
| --- | --- | --- |
| `type` | yes | Must be `"remote"` |
| `url` | yes | Remote MCP endpoint |
| `headers` | no | HTTP headers sent with requests; supports `{env:VAR}` |
| `enabled` | no | Enable/disable the server on startup |
| `oauth` | no | `false` to disable OAuth auto-detection, or an OAuth config object |
| `timeout` | no | Timeout in ms for MCP requests (default 5000) |

## OAuth

OAuth support is auto-detected for remote servers. For servers that use API keys or header auth instead, set `"oauth": false` to stop OpenCode attempting an OAuth flow.

Explicit OAuth configuration:

```jsonc
{
  "mcp": {
    "my-oauth-server": {
      "type": "remote",
      "url": "https://mcp.example.com/mcp",
      "oauth": {
        "clientId": "{env:MY_MCP_CLIENT_ID}",
        "clientSecret": "{env:MY_MCP_CLIENT_SECRET}",
        "scope": "tools:read tools:execute",
        "callbackPort": 19876,
        "redirectUri": "http://127.0.0.1:19876/mcp/oauth/callback"
      }
    }
  }
}
```

| Field | Description |
| --- | --- |
| `clientId` | OAuth client ID; if omitted, dynamic client registration (RFC 7591) is attempted |
| `clientSecret` | Client secret if the authorization server requires it |
| `scope` | Scopes to request |
| `callbackPort` | Local callback server port (default 19876); shorthand for `redirectUri` |
| `redirectUri` | Full redirect URI (default `http://127.0.0.1:19876/mcp/oauth/callback`) |

OAuth credentials are stored outside project config. Manage them with:

```bash
opencode mcp auth <name>      # authenticate
opencode mcp logout <name>    # remove stored credentials
opencode mcp debug <name>     # debug the OAuth connection
```

## Disabling servers and tools

- Temporarily disable a server without deleting its config: `"enabled": false`
- Disable every tool from one server while keeping others: `"tools": { "<server-name>": false }`
- Remote organizational defaults (`.well-known/opencode`) that ship disabled can be enabled locally with `"enabled": true`; local values override remote defaults

## CLI management

```bash
opencode mcp list                          # servers + connection status
opencode mcp add [name]                    # interactive add (remote URL or local command)
opencode mcp add foo --url https://...     # remote server with headers via --header KEY=VALUE
opencode mcp add bar --env KEY=VALUE       # local server with environment
opencode mcp auth [name]                   # OAuth flow for a remote server
opencode mcp logout [name]                 # remove OAuth credentials
opencode mcp debug <name>                  # OAuth / connection diagnostics
```

TUI: `/mcp` opens the server list; `space` toggles servers in the MCP dialog.

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Server never connects | `opencode mcp list` for status; check `command` path or `url`; raise `timeout` (default 5000ms is tight for slow servers) |
| 401/403 from a header-auth server | Set `"oauth": false` so OAuth doesn't interfere; verify header value and `{env:VAR}` expansion |
| OAuth login loop or failure | `opencode mcp debug <name>`; check `clientId`/`clientSecret`; ensure the callback port isn't taken |
| Env var empty in headers | `{env:VAR}` reads the environment of the opencode process — export it in the shell/profile that launches opencode |
| Too much context consumed | Disable unused servers (`enabled: false`) or gate specific server tools (`tools: { name: false }`) |
