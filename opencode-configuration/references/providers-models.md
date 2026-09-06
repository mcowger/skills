# OpenCode Providers, Custom Models & Model Management

Reference for `provider` configuration, model metadata, and provider visibility. Source of truth: `packages/core/src/v1/config/provider.ts`.

## The Two Golden Rules

1. **`enabled_providers`** (top-level array): when set, ONLY the listed providers load. Every auto-discovered provider (anthropic, openai, google, ...) is ignored.
2. **`whitelist`** (per-provider array): for custom providers, models appear in the `/models` selector only if their IDs are whitelisted. (For built-in providers the catalog from models.dev is used; whitelist narrows it.)

## Provider Config Shape

```jsonc
{
  "provider": {
    "<provider-id>": {
      "name": "Display Name",                    // shown in the UI
      "npm": "@ai-sdk/openai-compatible",        // AI SDK package implementing the provider
      "id": "optional-internal-id",
      "api": "optional-api-variant",
      "env": ["SOME_ENV_VAR"],                   // env vars that provide credentials
      "options": { /* provider options, see below */ },
      "whitelist": ["model-id-1", "model-id-2"], // only these models are selectable
      "blacklist": ["model-id-3"],               // these are removed from the set
      "models": { /* model metadata overrides, see below */ }
    }
  }
}
```

- `whitelist` narrows to the listed models; `blacklist` then removes entries from that narrowed set. IDs are the same ones shown in the `/models` picker.
- Overriding a **built-in** provider (e.g. `anthropic`) only needs the fields you're changing — the rest comes from models.dev catalog data.

## Provider Options (`options`)

| Option | Description |
| --- | --- |
| `apiKey` | API key; supports `{env:VAR}` substitution |
| `baseURL` | Base URL override — proxies, gateways, self-hosted endpoints |
| `timeout` | Full-request timeout in ms (default 300000); `false` disables |
| `headerTimeout` | Timeout in ms waiting for response headers; `false` disables |
| `chunkTimeout` | Timeout in ms between streamed SSE chunks; aborts on silence |
| `setCacheKey` | Always set a prompt cache key (default false) |
| `enterpriseUrl` | GitHub Enterprise URL for copilot auth |
| *(others)* | Any provider-specific option (passed through to the AI SDK integration), e.g. `region`/`profile` for Bedrock, `plexusBaseURL` for custom gateways |

### Common provider-specific options

- **Amazon Bedrock**: `region`, `profile` (from `~/.aws/credentials`), `endpoint` (VPC endpoint). Auth precedence: bearer token → AWS credential chain.
- **Azure OpenAI**: resource name via `AZURE_RESOURCE_NAME` env var.
- **OpenAI-compatible local servers** (vLLM, LM Studio, LiteLLM, Ollama via `@ai-sdk/openai-compatible`): `baseURL` + optional `apiKey`.

## Model Metadata (`models`)

Use `models` to add custom models to a provider or override catalog metadata for known ones:

```jsonc
{
  "provider": {
    "local-vllm": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://127.0.0.1:8000/v1" },
      "whitelist": ["Qwen/Qwen2.5-Coder-32B-Instruct"],
      "models": {
        "Qwen/Qwen2.5-Coder-32B-Instruct": {
          "name": "Qwen 2.5 Coder 32B",
          "attachment": false,
          "reasoning": false,
          "temperature": true,
          "tool_call": true,
          "limit": { "context": 65536, "input": 60000, "output": 8192 },
          "cost": { "input": 0.0, "output": 0.0 }
        }
      }
    }
  }
}
```

| Model field | Description |
| --- | --- |
| `id` | Override the model ID sent to the API (e.g. a Bedrock ARN) |
| `name` | Display name |
| `attachment` | Model accepts image attachments |
| `reasoning` | Model emits reasoning tokens |
| `temperature` | Model supports temperature |
| `tool_call` | Model supports tool calling |
| `interleaved` | Interleaved thinking support — bool, field name (`reasoning`, `reasoning_content`, `reasoning_text`), or `{ field }` |
| `limit.context` / `limit.input` / `limit.output` | Context window and output limits (tokens) |
| `cost.input` / `cost.output` / `cost.cache_read` / `cost.cache_write` | Per-token costs; optional `context_over_200k` block for long-context pricing |
| `modalities.input` / `modalities.output` | Arrays of `text`, `audio`, `image`, `video`, `pdf` |
| `status` | `alpha` \| `beta` \| `deprecated` \| `active` |
| `variants` | Variant-specific config, e.g. `{ "high": { "disabled": false } }` |
| `options` / `headers` | Per-model request options / headers |
| `provider.npm` / `provider.api` | Per-model provider package override |
| `release_date`, `family`, `experimental` | Catalog metadata |

## Model Selection Fields

| Field | Description |
| --- | --- |
| `model` | Default model, `provider/model-id` format |
| `small_model` | Cheap model for titles/summaries; default: cheapest from same provider, else `model` |
| Agent-level `model` | Per-agent override (`agent.<name>.model`) |
| Command-level `model` | Per-command override (`command.<name>.model`) |
| `variant` | Model variant (also settable per agent/command), cycled with `ctrl+t` in the TUI |

## Authentication

```bash
opencode auth login            # interactive provider picker (alias: opencode providers login)
opencode auth login anthropic  # skip the picker
opencode auth logout openai    # remove stored credentials
opencode providers list        # show credential status per provider
```

- In the TUI: `/connect` opens the same flow; supports API key entry and OAuth (e.g. Claude Pro/Max browser flow).
- Credentials are stored in `~/.local/share/opencode/auth.json` (never commit it).
- Some providers accept env vars instead (e.g. `AWS_PROFILE`, `AZURE_RESOURCE_NAME`, `CLOUDFLARE_ACCOUNT_ID`).
- OpenCode Zen / OpenCode Go are first-party model gateways — connect via `/connect` and key from opencode.ai/auth.

## Verification

```bash
opencode models                # list every model visible under the current config
opencode models plexus         # filter to one provider
opencode models --verbose      # include metadata like costs and limits
opencode models --refresh      # refresh the models.dev catalog cache
```

If a model is missing from this list, check `enabled_providers`, `whitelist`, and `blacklist` before anything else.
