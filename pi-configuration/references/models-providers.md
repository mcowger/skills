# Pi Providers & Models Reference

Configuration for `~/.pi/agent/models.json` (global-only; reloads when `/model` opens) and
`~/.pi/agent/auth.json` credentials.

## Top-level schema

```json
{
  "providers": {
    "<provider-id>": {
      "baseUrl": "https://...",
      "api": "openai-completions",
      "apiKey": "$ENV_VAR",
      "headers": {},
      "authHeader": false,
      "models": [],
      "modelOverrides": {}
    }
  }
}
```

Supported API types in models.json: `openai-completions`, `openai-responses`,
`anthropic-messages`, `google-generative-ai`.
Extensions add: `azure-openai-responses`, `openai-codex-responses`, `mistral-conversations`,
`google-vertex`, `bedrock-converse-stream`.

## Provider fields

| Field | Required | Meaning |
|---|---|---|
| `name` | no | Display name |
| `baseUrl` | usually | API endpoint; required for custom models |
| `api` | usually | API implementation (inherited by models) |
| `apiKey` | no | Literal / env reference / `!command` |
| `oauth` | no | `"radius"` for dynamic Radius providers |
| `headers` | no | Provider-wide headers |
| `authHeader` | no | Adds `Authorization: Bearer <apiKey>` |
| `compat` | no | Provider compatibility settings |
| `models` | no | Custom model definitions |
| `modelOverrides` | no | Override built-in/extension models by ID |

A provider needs an auth config before models appear in `/model` — use a dummy key
(`"apiKey": "ollama"`) for auth-less local servers.

## Secret expansion (apiKey, headers)

- `$VAR` and `${A}_${B}` env interpolation (works inside literals)
- `!command` executes a command, trimmed stdout becomes the value
  (e.g. `"!op read op://vault/item/credential"`, `"!security find-generic-password -ws 'anthropic'"`)
- `$$` → literal `$`; `$!` → literal `!`
- Plain text without `$`/`!` is literal
- `!command` results resolve at request time (no TTL/caching)

## Model fields

| Field | Default | Meaning |
|---|---|---|
| `id` | required | API model identifier |
| `name` | `id` | Display name and matching alias |
| `api` | provider `api` | Per-model API override |
| `baseUrl` | provider URL | Model-specific endpoint |
| `reasoning` | `false` | Extended thinking support |
| `thinkingLevelMap` | provider defaults | Map Pi levels → provider values; `null` hides a level |
| `input` | `["text"]` | E.g. `["text","image"]` |
| `contextWindow` | `128000` | Context size |
| `maxTokens` | `16384` | Output limit |
| `cost` | zeros | USD per 1M tokens: `input`/`output`/`cacheRead`/`cacheWrite` |
| `samplingParams` | unset | Free-form request params for OpenAI-compatible APIs |
| `headers` | unset | Model-specific headers |
| `compat` | provider `compat` | Per-model compatibility |

Cost tiers switch price on the entire request past a threshold:

```json
{ "cost": { "input": 5, "output": 30,
    "tiers": [ { "inputTokensAbove": 272000, "input": 10, "output": 45 } ] } }
```

## Compatibility (`compat`) fields

OpenAI-compatible:
```json
{ "compat": { "supportsDeveloperRole": false, "supportsReasoningEffort": false,
  "supportsUsageInStreaming": false, "maxTokensField": "max_tokens",
  "requiresToolResultName": true, "requiresAssistantAfterToolResult": true,
  "requiresThinkingAsText": true, "thinkingFormat": "qwen",
  "thinkingTokenBudgetField": "thinking_token_budget",
  "cacheControlFormat": "anthropic", "supportsStrictMode": true } }
```

Anthropic:
```json
{ "compat": { "supportsEagerToolInputStreaming": false, "supportsLongCacheRetention": true,
  "supportsCacheControlOnTools": true, "forceAdaptiveThinking": true,
  "allowEmptySignature": true, "supportsStrictTools": true, "supportsMidConvoEffort": true } }
```

## Overriding built-in providers

```json
{ "providers": { "anthropic": { "baseUrl": "https://my-proxy.example.com/v1" } } }
```

- `models` merge with built-ins; matching IDs replace built-in entries.
- `modelOverrides` patches metadata without redefining the catalog:

```json
{ "providers": { "openrouter": { "modelOverrides": {
  "anthropic/claude-sonnet-4": { "name": "Claude Sonnet 4 via Bedrock",
    "compat": { "openRouterRouting": { "only": ["amazon-bedrock"] } } } } } } }
```

## Complete local-provider example

```json
{
  "providers": {
    "local-vllm": {
      "baseUrl": "http://127.0.0.1:8000/v1",
      "api": "openai-completions",
      "apiKey": "none",
      "models": [{
        "id": "Qwen/Qwen2.5-Coder-32B-Instruct",
        "name": "Qwen 2.5 Coder 32B",
        "contextWindow": 65536,
        "maxTokens": 8192,
        "reasoning": false,
        "input": ["text"],
        "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
      }]
    }
  }
}
```

## Credentials (`auth.json`)

`~/.pi/agent/auth.json` (mode 0600). Resolution order per provider:

1. CLI `--api-key`
2. `auth.json`
3. Provider environment variable
4. `models.json` `apiKey`

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "cloudflare-ai-gateway": { "type": "api_key", "key": "$CLOUDFLARE_API_KEY",
    "env": { "CLOUDFLARE_ACCOUNT_ID": "acct", "CLOUDFLARE_GATEWAY_ID": "gw" } }
}
```

Credential-scoped `env` overrides process env for that credential only. Manage via `/login` / `/logout`.

## Extension providers

Use an extension (`pi.registerProvider`) when the provider requires custom OAuth, nonstandard
streaming, dynamic model discovery, or provider-specific request logic:

```typescript
pi.registerProvider("my-provider", {
  name: "My Provider",
  baseUrl: "https://api.example.com",
  apiKey: "$MY_API_KEY",
  api: "openai-completions",
  models: [{ id: "my-model", name: "My Model", reasoning: false, input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000, maxTokens: 4096 }]
});
```

`models.json` can override extension-registered providers/models. Example extensions live in the
pi repo at `packages/coding-agent/examples/extensions/custom-provider-*/`.
