# OMP MCP Servers Reference (`mcp.json`)

Model Context Protocol (MCP) server configuration, transports, authentication, and management in Oh My Pi. Schema: `packages/coding-agent/src/config/mcp-schema.json`.

## File Locations & Precedence

Primary OMP-native files:

1. **Project**: `<cwd>/.omp/mcp.json` — keyed to the working directory, applies under every profile
2. **User (default profile)**: `~/.omp/agent/mcp.json`
3. **User (named profile)**: `~/.omp/profiles/<name>/agent/mcp.json`

Compatibility aliases OMP also reads (but does not write): `.omp/.mcp.json`, `~/.omp/agent/.mcp.json`. Root fallback files `mcp.json` / `.mcp.json` are portable across clients.

Discovery also translates these tool-native sources:

- Claude Code: `~/.claude.json`, `~/.claude/mcp.json`, project `.claude/.mcp.json` / `.claude/mcp.json`
- Codex: `~/.codex/config.toml` and `.codex/config.toml` (`[mcp_servers.*]`)
- Gemini CLI: `~/.gemini/settings.json` and `.gemini/settings.json`
- OpenCode: `~/.config/opencode/opencode.json` and project-root `opencode.json`
- Cursor: `~/.cursor/mcp.json` and `.cursor/mcp.json`
- Windsurf: `~/.codeium/windsurf/mcp_config.json` and `.windsurf/mcp_config.json`
- VS Code: project-only `.vscode/mcp.json` (`mcp.servers`)
- installed Claude marketplace plugins and OMP extension packages

For Claude Code, Codex, Gemini CLI, Cursor, and Windsurf, the project entry precedes its same-named user entry, so a project `enabled: false` suppresses a same-named user server. Project config loading can be disabled with `mcp.enableProjectConfig: false`.

**Profiles.** A named profile's user scope resolves to the profile's agent directory, so it sees only its own user servers — never the default profile's `~/.omp/agent/mcp.json`. Add a server under a profile by launching with `omp --profile <name>` and running `/mcp add` → User level, or by editing `~/.omp/profiles/<name>/agent/mcp.json` directly.

---

## Top-Level Schema & Server Controls

```json
{
  "$schema": "https://raw.githubusercontent.com/can1357/oh-my-pi/main/packages/coding-agent/src/config/mcp-schema.json",
  "mcpServers": {
    "server-a": { "type": "http", "url": "https://example.com/mcp" },
    "server-b": { "enabled": false }
  },
  "disabledServers": ["server-b"],
  "enabledServers": ["server-a"]
}
```

- `$schema` — optional editor autocomplete/validation (OMP writes it automatically when `/mcp add` or another config-writing flow updates an OMP-managed file).
- `mcpServers` — map of server name → config. Names are up to 100 chars of letters, numbers, `_`, `-`, `.` (the runtime writer also allows `:`; the bundled schema does not, so a namespaced entry may validate at runtime while an editor warns).
- `disabledServers` — active-profile user denylist; hides a discovered server by name regardless of its source `enabled` value. Highest precedence.
- `enabledServers` — active-profile user allowlist; force-enables a same-named entry whose source says `enabled: false`. `disabledServers` still wins.

### Shared server fields (all transports)

- `enabled?: boolean` — skip this server when `false`, unless the allowlist names it.
- `timeout?: number` — MCP request timeout in ms; `0` disables client-side timeouts.
- `requestIdFormat?: "number" | "string"` — outgoing JSON-RPC id encoding; default per-transport integers. `"string"` uses collision-resistant snowflake ids. OMP-native only (ignored on imported tool configs).
- `auth?: { ... }` — stored-credential metadata (managed injection is implemented for OAuth).
- `oauth?: { ... }` — explicit OAuth client and callback settings.

`OMP_MCP_TIMEOUT_MS` has process-wide precedence over every per-server `timeout`. Set it to `0` to disable client-side timeouts, or a positive ms value. If unset or invalid, OMP uses the server value, then the 30-second default.

---

## Transports

### 1. Streamable HTTP (`type: "http"`)
Recommended for remote services and proxies. Requires `url`, optional `headers`.

```json
{
  "github": {
    "type": "http",
    "url": "https://api.githubcopilot.com/mcp/",
    "headers": { "Authorization": "Bearer sk-YourTokenHere" },
    "timeout": 60000
  }
}
```

### 2. stdio (`type: "stdio"`)
Default when `type` is omitted. Requires `command`, optional `args`, `env`, `cwd`.

```json
{
  "sqlite": {
    "type": "stdio",
    "command": "uvx",
    "args": ["mcp-server-sqlite", "--db-path", "/home/user/mydb.sqlite"],
    "env": { "DEBUG": "1" },
    "cwd": "/home/user"
  }
}
```

### 3. SSE (`type: "sse"`)
Legacy remote transport kept for compatibility; prefer `http` for new configs. Requires `url`, optional `headers`.

---

## Authentication

### OAuth

```json
{
  "slack": {
    "type": "http",
    "url": "https://mcp.slack.com/mcp",
    "oauth": {
      "clientId": "YOUR_SLACK_CLIENT_ID",
      "clientSecret": "YOUR_SLACK_CLIENT_SECRET",
      "scope": "channels:read",
      "callbackPort": 3000,
      "callbackPath": "/callback"
    },
    "auth": { "type": "oauth" }
  }
}
```

`oauth` fields: `clientId`, `clientSecret`, `scope`, `redirectUri`, `callbackPort` (1–65535), `callbackPath`, `prompt` (OAuth `prompt` parameter; default omitted unless the scope contains `offline_access`, which sends `consent`; set `""` to always omit).

`auth` fields: `type` (`oauth` | `apikey`), `credentialId`, `tokenUrl`, `clientId`, `clientSecret`, `resource` (OAuth resource indicators).

Trigger or refresh authorization inside a session with `/mcp reauth <name>` (or remove with `/mcp unauth <name>`). Credentials live in the active profile's auth storage (`agent.db` or auth broker) and refresh automatically — never in a committed config file.

---

## Session Commands

MCP is managed through slash commands, not a standalone CLI command:

| Command | Effect |
| --- | --- |
| `/mcp add` | Guided server setup |
| `/mcp list` | Show servers and which config file each came from |
| `/mcp test <name>` | Test a single server |
| `/mcp reload` | Rediscover and reconnect all servers |
| `/mcp reconnect <name>` | Reconnect one server without rediscovering |
| `/mcp reauth <name>` / `/mcp unauth <name>` | Replace / remove managed OAuth credentials |
| `/mcp enable <name>` / `/mcp disable <name>` | Toggle a server |
| `/mcp resources`, `/mcp prompts`, `/mcp notifications` | Inspect non-tool MCP capabilities |

`/mcp enable` and `/mcp disable` update `enabled` directly only for OMP-owned writable files; for another tool's config they maintain the user-level allowlist/denylist instead.

---

## Related Settings

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `mcp.enableProjectConfig` | boolean | `true` | Load project-scoped MCP config. |
| `mcp.renderMarkdownResults` | boolean | `true` | Render markdown in MCP results. |
| `mcp.notifications` | boolean | `false` | Surface server notifications. |
| `mcp.notificationDebounceMs` | number | `500` | Notification debounce window. |

---

## Troubleshooting

- **Server does not connect** — run `/mcp test <name>`; check the binary/image exists, required env vars are set, and the URL is reachable.
- **Server not discovered** — run `/mcp list`. Project loading may be off (`mcp.enableProjectConfig: false`) or a user-level `disabledServers` entry may suppress it by name.
- **Editor rejects a valid name** — the bundled schema omits `:` from its name pattern, so a runtime-valid namespaced entry (e.g. `cloudflare:cloudflare-api`) may still warn in an editor.
- **Malformed JSON** — the provider contributes no entries and records a discovery warning rather than failing the session; fix the shape, then `/mcp reload`.
