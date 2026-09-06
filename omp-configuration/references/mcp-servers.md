# OMP MCP Servers Reference (`mcp.json`)

This reference covers Model Context Protocol (MCP) server configuration, transports, authentication, and management in Oh My Pi.

## File Locations & Precedence

1. **Project-Scoped**: `<cwd>/.omp/mcp.json` (evaluated under the project repository)
2. **User-Scoped (Default Profile)**: `~/.omp/agent/mcp.json`
3. **User-Scoped (Named Profile)**: `~/.omp/profiles/<name>/agent/mcp.json`
4. **Fallback / Universal**: `<cwd>/mcp.json` or `<cwd>/.mcp.json`

Cross-tool discovery also translates configs from `.claude/mcp.json`, `.cursor/mcp.json`, and `opencode.json`.

---

## MCP Transports

### 1. Streamable HTTP (`type: "http"`)
Recommended for remote services and proxies.

```json
{
  "github": {
    "type": "http",
    "url": "https://api.githubcopilot.com/mcp/",
    "headers": {
      "Authorization": "Bearer sk-YourTokenHere"
    },
    "timeout": 60000
  }
}
```

### 2. Standard Input/Output (`type: "stdio"`)
Used for local command-line tools and node packages. `type` defaults to `stdio` if omitted.

```json
{
  "sqlite": {
    "type": "stdio",
    "command": "uvx",
    "args": ["mcp-server-sqlite", "--db-path", "/home/matt.cowger/mydb.sqlite"],
    "env": {
      "DEBUG": "1"
    },
    "cwd": "/home/matt.cowger"
  }
}
```

### 3. Server-Sent Events (`type: "sse"`)
Legacy remote transport.

```json
{
  "remote-agent": {
    "type": "sse",
    "url": "https://mcp.internal.example.com/sse",
    "headers": {
      "X-Api-Key": "secret"
    }
  }
}
```

---

## Top-Level Schema & Server Controls

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "server-a": { ... },
    "server-b": { ... }
  },
  "disabledServers": ["server-b"],
  "enabledServers": ["server-a"]
}
```

- `disabledServers`: Denylist of server names to ignore.
- `enabledServers`: Allowlist to force-enable servers even if marked `enabled: false`.
- `timeout`: Per-server timeout in milliseconds (`0` = no timeout).
- `OMP_MCP_TIMEOUT_MS`: Global environment variable that overrides all server timeouts.

---

## OAuth Authentication

For HTTP servers using OAuth 2.0:

```json
{
  "slack": {
    "type": "http",
    "url": "https://mcp.slack.com/mcp",
    "oauth": {
      "clientId": "YOUR_SLACK_CLIENT_ID",
      "clientSecret": "YOUR_SLACK_CLIENT_SECRET",
      "callbackPort": 3000,
      "callbackPath": "/callback"
    },
    "auth": {
      "type": "oauth",
      "tokenUrl": "https://slack.com/api/oauth.v2.user.access",
      "clientId": "YOUR_SLACK_CLIENT_ID",
      "clientSecret": "YOUR_SLACK_CLIENT_SECRET"
    }
  }
}
```

Trigger authorization inside a session:
```bash
/mcp reauth slack
```
Tokens are securely stored in `~/.omp/agent/agent.db` and refreshed automatically.
