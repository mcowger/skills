# pi-mcp-adapter Reference

Token-efficient MCP adapter for Pi: [nicobailon/pi-mcp-adapter](https://github.com/nicobailon/pi-mcp-adapter).
One `mcp()` proxy tool (~200 tokens) instead of hundreds of tool definitions; servers start lazily.
Core Pi has no native MCP support — this package provides it.

**Install:** `pi install npm:pi-mcp-adapter` → restart Pi. Pin for reproducibility
(`npm:pi-mcp-adapter@2.26.1` etc. — real version-specific regressions have occurred, e.g. issue #415).
**Always `/reload`** after editing any mcp.json file.

## Config files & precedence (later wins)

1. `~/.config/mcp/mcp.json` — user-global shared
2. `~/.agents/mcp.json` — tool-agnostic global
3. `~/.agents/mcp/mcp.json` — tool-agnostic global
4. `<pi agent dir>/mcp.json` (`~/.pi/agent/mcp.json`) — Pi global override, compat imports, adapter-owned server edits (`directTools` persisted here for shared servers)
5. `.mcp.json` — project shared
6. `.pi/mcp.json` — Pi project override; `/mcp disable|enable <server>` writes only `{ "disabled": ... }` here, then `/reload`

Host configs (cursor, claude-code, claude-desktop, codex, opencode, windsurf, vscode) are
**not loaded by default** (`settings.hostConfigDiscovery: "off"` | `"prompt"` | `"on"`).
Adopt them via `/mcp setup` (diff preview), `pi-mcp-adapter init`, or `init --discover-host-configs`.
Discovery never writes to external host files or silently runs their commands.

## Root schema

```json
{ "mcpServers": {}, "imports": ["opencode"], "settings": {}, "claudePlugins": [] }
```

(`mcp-servers` accepted as input alias for `mcpServers`; `imports` are compatibility imports.)

## Server fields

### Transport & process

| Field | Notes |
|---|---|
| `command` | stdio executable; exclusive with `url` and `socket` |
| `args`, `env`, `cwd` | stdio params; interpolation `${VAR}`, `$env:VAR`, `~` |
| `inheritEnv` | default `true`; `false` excludes inherited host env (not sandboxing) |
| `url` | Streamable HTTP with SSE fallback; interpolation |
| `headers` | HTTP headers; interpolation and `!command` secrets |
| `socket` | explicit `rmcp-mux` Unix socket |
| `requestHeadersCommand` | runs per HTTP request returning derived headers |
| `httpTransport` | `"streamable-http"` \| `"sse"` (Agent Plugin declarations; disables client fallback) |
| `pluginDataDir` / `literalEnv` | Agent Plugin behaviors |

Secret commands (`!cmd`) in `env`/`headers`/`bearerToken`/`oauth.clientSecret`: no stdin,
stderr suppressed, 1 MiB stdout cap, 10 s timeout, non-empty trimmed output. `!!` = literal `!`.

### Auth

| Field | Notes |
|---|---|
| `auth` | `"oauth"` \| `"bearer"` \| `false`. Omitted with `url`: OAuth auto-detected unless nonempty `headers` |
| `bearerToken` | literal / interpolation / `!command` |
| `bearerTokenEnv` | env var name |
| `bearerTokenStore` | `true`: read from OS keyring (requires `auth: "bearer"`) |
| `oauth` | object (below) or `false` |

```json
{ "oauth": { "grantType": "authorization_code",   // or "client_credentials" (no browser)
  "clientId": "pre-registered-client",            // else Dynamic Client Registration
  "clientSecret": "secret", "scope": "read write",
  "authorizationParams": { "prompt": "consent" }, // cannot override client_id/redirect_uri/scope/state/code_challenge/response_type/resource
  "redirectUri": "http://localhost:3118/callback", // {port} supported; HTTPS callbacks = manual completion
  "clientName": "Pi Coding Agent", "clientUri": "...", "logoUri": "https://...",
  "authServerMetadataUrl": "https://.../.well-known/oauth-authorization-server",
  "skipIssuerMetadataValidation": false } }
```

OAuth: official MCP SDK flow (OAuth 2.1, PKCE S256, protected-resource discovery, DCR fallback,
token refresh, state/CSRF). Loopback callback, OS-assigned port, 5-min pending timeout;
pre-registered clients default to port 19876 (`MCP_OAUTH_CALLBACK_PORT`).

### Runtime behavior

| Field | Default | Notes |
|---|---|---|
| `lifecycle` | `lazy` | `lazy` / `eager` / `keep-alive` / `lazy-keep-alive` |
| `idleTimeout` | global (10 min) | `0` disables idle disconnect |
| `requestTimeoutMs` | SDK default | `<=0`/omitted = SDK default |
| `protocolVersion` | `"legacy"` | also `"auto"`, `"2026-07-28"` |
| `exposeResources` | `true` | MCP resources as generated `read_<resource>` tools |
| `directTools` | `false` | `true` or string[] |
| `toolPrefix` | inherits `server` | `server`/`short`/`none`/`mcp` |
| `includeTools` / `excludeTools` | — | patterns; exclude applied after include; affects direct tools, proxy search, panel |
| `searchKeywords` | — | tool/glob → extra search terms (ranking only) |
| `approveTools` | `false` | `true` or patterns → interactive approval before call |
| `debug` | `false` | show stdio server stderr |
| `trace` | `false` | metadata-only protocol tracing |
| `disabled` | `false` | only literal `true` disables |

Lifecycle semantics: `lazy` connect-on-first-use + idle disconnect; `eager` connect at startup,
no auto-reconnect/idle timeout unless set; `keep-alive` startup connect + periodic refresh;
`lazy-keep-alive` first-use connect then resident.

## Global `settings` (inside mcp.json)

| Setting | Default | Notes |
|---|---|---|
| `toolPrefix` | `"server"` | direct tool naming: `<server>_<tool>`; `short` strips trailing `-mcp`; `mcp` → `mcp__server__tool`; `none` bare |
| `showStatusIcon` | `true` | status icon in MCP status text |
| `mcpFooterStatus` | `"full"` | `full`/`compact`/`off` |
| `toolResultRendering` | `"compact"` | or legacy `"boxed"` |
| `collapsedResultLines` | 1 compact / 3 boxed | allowed 1–3 |
| `notifyOnStartupConnect` | `true` | |
| `hostConfigDiscovery` | `"off"` | `off`/`prompt`/`on` |
| `agentPluginPaths` | empty | Agent Plugins 1.0 dirs (relative to project cwd); servers prefixed `plugin__server`; `${PLUGIN_ROOT}`/`${PLUGIN_DATA}` expanded |
| `idleTimeout` | 10 minutes | global default, `0` disables |
| `requestTimeoutMs` | SDK default | global live-request timeout |
| `directTools` | `false` | global default for all servers |
| `strictDirectToolArguments` | `false` | validate args; recover one JSON-string layer |
| `directToolResultDetails` | `"lean"` | or `"bounded"` |
| `warnOnLargeDirectTools` | `true` | warn at ≥75 resolved direct tools |
| `scriptMode` | `true` | register the `mcpScript` tool |
| `approveTools` | `false` | global approval gate / glob array |
| `disableProxyTool` | `false` | hide `mcp` only when direct tools fully available |
| `freezeDirectTools` | `false` | don't rebuild direct registrations on metadata update |
| `autoAuth` | `false` | auto-OAuth once on auth failure and retry |
| `sampling` / `elicitation` | enabled (UI) | let servers request model sampling / form elicitation; `samplingAutoApprove` needed headless |
| `outputGuard` | `true` | or object `{maxBytes: 51200, maxLines: 2000, detailsMaxBytes: 16384}`; `MCP_OUTPUT_GUARD=0` disables text guarding |
| `trace` | off | `{enabled, file, maxBytes: 262144, maxEvents: 10000}`; default path `.pi/mcp-traces/mcp-<ts>.jsonl`; never logs payloads/prompts/tokens |
| `authRequiredMessage` | built-in | custom message; `${server}` substituted |
| `oauthDir` | legacy | legacy plaintext import dir only |

## The `mcp()` proxy tool

Mode precedence: `action` > `tool` (call) > `connect` > `describe` > `instructions` > `search` > `server` > nothing (status).

- `mcp({})` — status: servers (`connected`/`cached`/`failed`/`needs-auth`/`not connected`/`disabled`), counts
- `mcp({server: "github"})` — list a server's tools + instructions preview
- `mcp({search: "screenshot", server?, limit?: 12, offset?: 0, includeSchemas?: true, regex?})` — searches names/described/`searchKeywords`; `details.nextOffset` for pagination
- `mcp({describe: "github_search_repositories"})` — full schema; hyphen/underscore fuzzy; ambiguous names need `server`
- `mcp({instructions: "github"})` — cached server instructions
- `mcp({connect: "github"})` — connect lazy server / refresh tools, resources, prompts, cache, direct tools (result may list `addedToolNames`)
- `mcp({tool: "github_search_repositories", args: {...} | "{...}"})` — call; gateway fields must be top-level (never nest under `args`)
- `mcp({action: "auth-start", server})` → returns auth URL; `mcp({action: "auth-complete", server, args: {redirectUrl}})` (headless flow: open URL locally, paste full callback URL)
- `mcp({action: "ui-messages"})` — accumulated messages from completed MCP UI sessions

## directTools

Promote MCP tools to first-class Pi tools (bypass the proxy): `true`, or string[] of
**original** MCP tool names. Global default via `settings.directTools`.

- Cache-driven: reconstructed from `~/.pi/agent/mcp-cache.json` (7-day default validity); new
  servers are proxy-only until one successful connection; `/mcp reconnect <server>` forces discovery
- Built-in name collisions (`read`, `bash`, `edit`, `mcp`) and duplicate names skipped
- ~150–300 prompt tokens per direct tool; keep 5–20 per server; ≥75 triggers a warning
- Env override: `MCP_DIRECT_TOOLS=chrome-devtools,github` or `github/search_repositories,...` or `__none__`

Strict args checking: `settings.strictDirectToolArguments`.

## Commands

| Command | Behavior |
|---|---|
| `/mcp` (`/pi-mcp` alias) | Server panel (TUI) / status text (headless) |
| `/mcp status` `tools` `prompts` | Text views; prompts lists MCP-prompt slash commands |
| `/mcp reconnect [server]` | Force-close + reconnect, refresh catalog |
| `/mcp setup` | Guided setup: pick project `.mcp.json` vs global, imports adoption with diffs, scaffold empty config, known presets (DeepWiki, Context7, Parallel Search, Notion, GitHub, Chrome DevTools), RepoPrompt |
| `/mcp disable <server>` / `/mcp enable <server>` | Writes only `disabled` to `.pi/mcp.json`; **`/reload` required** |
| `/mcp logout <server>` | Delete stored OAuth creds + disconnect |
| `/mcp token status/remove <server>` | Bearer-store management (`set` intentionally refused in UI — no masked input) |
| `/mcp-auth` | OAuth server picker (interactive) |
| `/mcp-auth <server>` | Run OAuth flow; auto-reconnects on success |

No `/mcp add` or `/mcp auth` — adding happens via `/mcp setup`, OAuth via `/mcp-auth`.

Panel keys: ↑/↓ navigate · Space toggle directness · Enter expand/toggle/auth · Ctrl+A OAuth ·
Ctrl+R reconnect · Ctrl+D enable/disable · Ctrl+Y copy failure · `?` description search ·
Ctrl+S save (remap `mcp.panel.save`) · Esc close (60 s inactivity timeout). Panel direct-tool
toggles apply immediately; broader changes invoke Pi reload.

## Credential storage

OS keyring via `@napi-rs/keyring` (macOS Keychain, Windows Credential Manager, Linux Secret Service):

- OAuth: service `pi-mcp-adapter.oauth`, account `sha256-<sha256(server-name)>`, URL-bound —
  changing a server's URL invalidates stored creds
- Bearer store: service `pi-mcp-adapter.bearer`, URL-bound record `{token, serverUrl}`;
  CLI: `pi-mcp-adapter token set|status|remove <server>` — token must come via masked prompt
  or stdin (`printf '%s' "$TOKEN" | pi-mcp-adapter token set github`), never as argv; needs Node ≥22.18
- Legacy plaintext `~/.pi/agent/mcp-oauth/sha256-<hash>/tokens.json` (or `$MCP_OAUTH_DIR` /
  `settings.oauthDir`) is imported **once** into the keyring and deleted
- Fails closed when no keyring — no silent plaintext fallback; in-process credential cache
  disabled with `PI_MCP_ADAPTER_DISABLE_AUTH_CACHE=1`
- Public API: `pi-mcp-adapter/oauth` exports `getMcpOAuthTokensForUrl` / `updateMcpOAuthTokensForUrl`

## Programmatic API

```ts
import { createMcpAdapter } from "pi-mcp-adapter";
const extension = createMcpAdapter({ config: { mcpServers: { docs: { url: "...", lifecycle: "eager" } } } });
// or { configPath: "..." }
```

Supplied `config` is an isolated snapshot: no ambient files, imports, project config, or
`--mcp-config` merging; cloned, never mutated. Unavailable in that mode: setup, ambient discovery,
no-arg OAuth picker, project `disabled` overrides. Default export = normal file-based adapter.

Runtime (non-persisted, proxy-only, session-scoped) registration also available:

```ts
import { registerMcpServer } from "pi-mcp-adapter";
const reg = registerMcpServer({ pi, name: "acme__docs", definition: { url: "..." } });
await reg.dispose();
```

## Adapter-owned files

| File | Purpose |
|---|---|
| `~/.pi/agent/mcp.json` | Pi global config/override/imports |
| `~/.pi/agent/mcp-cache.json` | Tool/resource/prompt/instructions cache + config hashes |
| `~/.pi/agent/mcp-onboarding.json` | First-run / setup state |
| `.pi/mcp.json` | Project Pi override (disable flags) |
| `.pi/mcp-traces/*.jsonl` | Optional metadata-only traces |
| `~/.pi/agent/mcp-oauth/...` | Legacy import locations only |

## Troubleshooting

| Symptom | Fix |
|---|---|
| Changes not applying | `/reload`; `/mcp reconnect <server>` for live refresh |
| `spawn ENOENT` on stdio server | Check `cwd` exists (misleading error, issue #442), command on PATH, `"debug": true` for stderr, env interpolation vars exist |
| OAuth headers ignored / DCR triggered | Nonempty `headers` suppresses OAuth auto-detect; set `auth: false` explicitly if appropriate (issue #158) |
| "OAuth secure credential storage is unavailable" | Headless Linux lacks keyring; adapter fails closed. Use `bearerTokenEnv` instead (issues #218, #248: tmux/WSL revoked session keyrings — recovery needs `keyctl` + `node` on PATH) |
| Auth succeeds but server stays failed | Should auto-reconnect now (issue #171); manual `/mcp reconnect <server>` |
| `-32000 Server not initialized` after remote restart | Stale-session recovery reconnects once/retries once (issue #184) |
| Keep-alive refresh failures on slow servers | Refresh capped at 5 s; consider `lazy`/`lazy-keep-alive` (issue #400) |
| "MCP not initialized" | Transient startup failure; fix config and retry via `mcp(...)` (issue #428) |
| Too many direct tools | Use proxy-only / `directTools: false` / arrays / `includeTools` (issue #240) |
| `npx --` wrapping broken | Use direct executable instead of wrapper tools like dotenv-cli (issue #15) |
| Endpoint "does not appear to speak MCP" | URL returns HTML — wrong endpoint; HTTP 503 = temporary, keep-alive retries with backoff |
| Windows native binding issues | `@napi-rs/keyring` absolute-path fallback (issue #230); large creds chunked for Credential Manager |
