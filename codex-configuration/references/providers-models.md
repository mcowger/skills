# Codex Providers, Model Catalog & Compatibility

Reference for `model_provider` selection, `[model_providers.*]` definitions, and how the model catalog works. Sources of truth: `codex-rs/model-provider-info/src/lib.rs` (provider metadata + validation), `codex-rs/model-provider/src/{provider,models_endpoint,auth}.rs` (runtime selection, `/models` refresh, auth resolution), `codex-rs/models-manager/src/manager.rs` (refresh strategies, cache, catalog merging).

## Selecting the active provider

```toml
model_provider = "custom"                          # key into [model_providers.*]
openai_base_url = "https://proxy.example.com/v1"   # ONLY the built-in openai endpoint
oss_provider = "ollama"                             # ollama | lmstudio (for --oss / --local-provider)
```

- `model_provider` names a key in the merged provider map (built-ins + your `[model_providers.*]`). Default `"openai"`.
- There is no provider allowlist/visibility list — exactly one provider is active per session.
- CLI: `--oss` routes to the OSS provider (`oss_provider` config or `--local-provider ollama|lmstudio`); `-m <slug>` picks the model within the active provider's catalog.

## Built-in providers

Compiled into the binary (`built_in_model_providers()`); only here so Codex works out-of-the-box. Users add third parties via `[model_providers.*]` — the codebase deliberately does not adjudicate which third parties to bundle.

| ID | Endpoint | Auth | Knows |
| --- | --- | --- | --- |
| `openai` | `https://api.openai.com/v1` (or `openai_base_url`) | `requires_openai_auth = true` (ChatGPT login / API key / PAT / agent identity); sends `version` header + `OpenAI-Organization`/`OpenAI-Project` from env | WebSockets, standalone web search, V2 remote compaction, attestation (ChatGPT auth) |
| `amazon-bedrock` / `amazon-bedrock-runtime` | Regional Mantle endpoint (derived; `base_url` = explicit endpoint override) | `aws` block only (SigV4) | Static catalogs: `openai.gpt-5.4/5.5/5.6-sol/terra/luna`, `openai.gpt-6-astra`; runtime adds `global.` + `us.` routing variants; text-only web search; multi-agent V1; default-tier only |
| `ollama` | `http://localhost:11434/v1` | none | Local OSS; `CODEX_OSS_PORT` / `CODEX_OSS_BASE_URL` (experimental env overrides) |
| `lmstudio` | `http://localhost:1234/v1` | none | Local OSS; dedicated load-model flow |

## Custom providers

```toml
[model_providers.custom]
name = "OpenAI via Plexus"
base_url = "https://plexus.home.cowger.us/v1"
wire_api = "responses"            # ONLY "responses" — "chat" is a hard error with a fix-up message
env_key = "AIHOME_API_KEY"        # env var holding the bearer token
env_key_instructions = "Get a key at https://..."   # shown when env_key is missing/empty
requires_openai_auth = false
supports_websockets = false
supports_standalone_web_search = false
```

### Rules (enforced at config load)

- **Reserved IDs**: `openai`, `ollama`, `lmstudio` cannot be redefined — rename yours (e.g. `openai-custom`). Error message says exactly this.
- **Bedrock pair are special**: `amazon-bedrock` / `amazon-bedrock-runtime` accept *partial* overrides — only `base_url`, `auth`, `http_headers` (merged), `aws.profile`, `aws.region`, `aws.credential_export`, `aws.auth_refresh`. Any other non-default field is an error. All other custom IDs are *insert-only* (ignored if the key already exists).
- **Empty names rejected** for non-Bedrock custom providers.

### Auth options — pick exactly one style

| Field | Meaning |
| --- | --- |
| `env_key` | Env var read at request time; missing/empty → actionable error naming the var (+ `env_key_instructions`). The standard choice — keeps secrets out of config |
| `auth = { command, args, timeout_ms, ... }` | Shell out to a command for a bearer token (`ModelProviderAuthInfo`) |
| `experimental_bearer_token` | Literal token in config. **Discouraged** for security; prefer `env_key` |
| `requires_openai_auth = true` | Force the `codex login` flow (ChatGPT/API key/PAT/agent identity). First-party path |
| `aws = { profile, region, credential_export = { command, args, timeout_ms }, auth_refresh = { command = "aws", ... } }` | Bedrock-only SigV4. `credential_export.command` must be absolute or a bare name and can't combine with `aws.profile`; `auth_refresh.command` must be exactly `aws`; `aws` can't combine with `env_key`/`experimental_bearer_token`/`auth`/`requires_openai_auth`/`supports_websockets` |

Validation also rejects: empty `auth.command`; `auth` combined with `env_key`/`experimental_bearer_token`/`requires_openai_auth`.

### Transport & resilience knobs

| Field | Default | Meaning |
| --- | --- | --- |
| `wire_api` | `responses` | Only valid value |
| `http_headers` | — | Static extra headers (openai built-in sends `version`; beware secret leakage into config) |
| `env_http_headers` | — | `{ "Header-Name": "ENV_VAR" }` — value read from env per request, skipped when unset/empty |
| `query_params` | — | Appended to the base URL |
| `request_max_retries` | 4 (cap 100) | HTTP request retries (5xx + transport; 429 *not* retried by default) |
| `stream_max_retries` | 5 (cap 100) | Streaming-reconnect attempts |
| `stream_idle_timeout_ms` | 300000 | Silence before a stream is treated as lost |
| `websocket_connect_timeout_ms` | 15000 | WebSocket connect timeout |

## How the model catalog works

Three sources, in increasing authority:

1. **Bundled fallback** compiled into the binary (offline-safe baseline).
2. **Live `/models` refresh** (default path): `GET {base_url}/models`, 5 s timeout, ETag revalidation. Results cache to `$CODEX_HOME/models_cache.json` (**TTL 300 s**, rejected on client-version mismatch) keyed by a sha256 of provider + auth identity (ChatGPT access tokens excluded so rotation doesn't invalidate; explicit bearer/command-auth headers *included* so key changes bust the cache). Strategies: `Online` (always fetch) / `Offline` (cache only) / `OnlineIfUncached`. First-party (ChatGPT/API-key) auth refreshes; bare `env_key` providers use the bundled catalog unless command-auth is present.
3. **Static override** via `model_catalog_json = "/abs/path.json"` — `{"models": [...]}` becomes the *authoritative in-memory* catalog: no fetch, no cache writes. **Applied on startup only**; per-thread `config` overrides are accepted but no-ops.

Merging: a ChatGPT-visible catalog (`visibility: "list"` + ChatGPT account) *replaces* the bundle; other catalogs *augment* it slug-by-slug. Picker presets sort by `priority`, filter by auth mode (`supported_in_api`, backend-only gating), and mark the default by picker visibility. `codex debug models` prints the active catalog.

### Slug resolution

Longest-prefix match (`gpt-5.6-luna-extra` → `gpt-5.6-luna` entry, slug preserved), plus a one-segment namespace retry (`custom/gpt-5.3-codex` → `gpt-5.3-codex`). Total miss → synthetic fallback entry (`used_fallback_model_metadata`, ~272k context) + warning. Bedrock keeps its own static `openai.gpt-*` slugs (no prefix games needed).

## Compatibility options (per catalog entry)

Each `ModelInfo` decides what Codex attempts for that model — this is the "compat" surface:

| Field | Effect |
| --- | --- |
| `shell_type` | `shell_command` vs `unified_exec` tool shape |
| `tool_mode` | `code_mode_only` gates non-code-mode tools |
| `use_responses_lite` | Slimmed Responses payloads |
| `supports_parallel_tool_calls` | Parallel fan-out allowed |
| `supports_image_detail_original` | Full-resolution images |
| `input_modalities` | e.g. `["text", "image"]` |
| `truncation_policy` | `{ mode = "tokens" \| "bytes", limit }` for tool outputs |
| `web_search_tool_type` / `apply_patch_tool_type` | `text` vs `text_and_image` / freeform variants |
| `supported_reasoning_levels` + `default_reasoning_level` | Effort ladder the model exposes (`low`…`ultra`; astra-class defaults `low`) |
| `support_verbosity` + `default_verbosity` | Whether `model_verbosity` applies |
| `default_reasoning_summary` | Summary style default |
| `context_window` / `max_context_window` / `auto_compact_token_limit` / `comp_hash` | Window + compaction behavior (`model_context_window` clamps to max) |
| `service_tiers` / `additional_speed_tiers` / `default_service_tier` | Priority/flex routing |
| `visibility` | `list` (picker-visible) vs hidden/backend-only (`supported_in_api`, auth-filtered) |
| `priority` / `minimal_client_version` | Picker order; clients older than the minimum hide the entry |
| `multi_agent_version` / `multi_agent_reasoning_effort` | Delegation protocol + effort |
| `model_messages` | Per-model prompt templates (instructions, approvals, collaboration modes, guardian v2, token budget) |

### Custom catalogs (the gateway pattern)

When the provider is a proxy that doesn't return a usable `/models` (or you want to pin the list), ship a static file:

```bash
codex debug models > /tmp/catalog.json   # capture the live shape once
# prune/edit entries, then pin it:
# model_catalog_json = "/home/user/.codex/model-catalog-proxy.json"
```

This machine does exactly that: `model_catalog_json` points at a chezmoi-tracked snapshot (`private_dot_codex/model-catalog-proxy.json`) so the picker is stable. **Refresh the snapshot when the gateway adds models** — a stale file hides new slugs and stale capability fields (reasoning ladder, context window, tool mode) cause subtle misbehavior. Keep the file's entries faithful to the live `/models` shape (same `ModelInfo` keys); unknown slugs fall back to synthetic metadata with a warning rather than failing.
