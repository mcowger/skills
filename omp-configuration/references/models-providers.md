# OMP Models & Providers Reference (`models.yml`)

Custom model definitions, local LLM integrations, OpenAI/Anthropic-compatible endpoints, discovery mechanisms, and secret resolution in `~/.omp/agent/models.yml`. Schema source: `packages/coding-agent/src/config/models-config-schema-bundle.ts`.

## File Location & Syntax

- Primary path: `~/.omp/agent/models.yml` (or `.yaml`). If both YAML files are missing but `models.json` exists, it is migrated to `models.yml` once.
- Top-level structure MUST contain only a `providers` mapping; unknown root keys fail schema validation.

```yaml
providers:
  <provider-id>:
    baseUrl: <url>
    apiKey: <key-or-command>
    api: <api-type>
    headers:
      <header-name>: <header-value>
    auth: apiKey                      # apiKey | none | oauth
    authHeader: true
    disableStrictTools: false
    discovery:
      type: <discovery-type>
      timeoutMs: 10000
    modelOverrides:
      <model-id>:
        name: <display-name>
    models:
      - id: <model-id>
        name: <display-name>
        api: <api-type>
        reasoning: false
        input: [text, image]
        contextWindow: <tokens>
        maxTokens: <tokens>
        cost:
          input: 0
          output: 0
          cacheRead: 0
          cacheWrite: 0
        compat:
          supportsStore: true
          supportsDeveloperRole: true
          supportsReasoningEffort: true
          maxTokensField: max_completion_tokens
```

---

## Provider-level Fields

| Field | Purpose |
| --- | --- |
| `baseUrl` | Provider API root. |
| `apiKey` | Literal key or `!command` (see below). Required unless `auth: none`. |
| `api` | Protocol for every model under the provider (overridable per model). |
| `headers` | Extra request headers. |
| `compat` | Per-API compatibility flags (see below). |
| `remoteCompaction` | Provider-native compaction baseline for eligible models. |
| `authHeader` | Send the key as an `Authorization` header. |
| `auth` | `apiKey` (default), `none`, or `oauth`. |
| `discovery` | Dynamic model discovery (see below). |
| `models` | Inline custom model definitions. |
| `modelOverrides` | Sparse patches applied to bundled models by id. |
| `disableStrictTools` | Set `true` for Anthropic-compatible endpoints that reject the strict field. |
| `guardrailIdentifier` / `guardrailVersion` / `guardrailTrace` | Amazon Bedrock guardrail id/version/trace (`enabled`, `disabled`, `enabled_full`). |
| `requestMetadata` | Bedrock invocation-log tags (max 16 entries). |
| `transport` | `pi-native` dispatches every model via an auth-gateway `POST /v1/pi/stream`; `baseUrl` must point at a compatible `omp auth-gateway` and `apiKey` is the gateway bearer. |

---

## Supported `api` Protocol Types

| API Protocol | Description | Common Backends |
| --- | --- | --- |
| `openai-completions` | Standard Chat Completions (`/v1/chat/completions`) | vLLM, LM Studio, Ollama, LiteLLM, Groq |
| `openai-responses` | OpenAI Responses API (`/v1/responses`) | OpenAI official, Ollama native, LiteLLM Responses |
| `openai-codex-responses` | OpenAI Codex streaming protocol | Codex-optimized endpoints |
| `azure-openai-responses` | Azure OpenAI Responses API | Azure OpenAI deployments |
| `anthropic-messages` | Anthropic Messages API (`/v1/messages`) | Anthropic official, Bedrock, Anthropic proxies |
| `bedrock-converse-stream` | AWS Bedrock Converse Stream | Bedrock Claude & Llama models |
| `google-generative-ai` | Google AI Studio Gemini API | Gemini Developer API |
| `google-gemini-cli` | Gemini CLI OAuth surface | Gemini CLI accounts |
| `google-vertex` | Google Cloud Vertex AI protocol | Vertex AI Gemini models |

---

## Model Definition & Override Fields

Both `models[]` entries and `modelOverrides.<id>` accept the same sparse shape (an override omits `id`; nothing is required):

| Field | Notes |
| --- | --- |
| `id` | Required for a `models[]` entry; non-empty. |
| `name` | Display name. |
| `api`, `baseUrl` | Per-model protocol/root override (definition only). |
| `reasoning` | Boolean reasoning support. |
| `thinking` | Thinking-control config: `mode` (`effort`, `budget`, `google-level`, `anthropic-adaptive`, `anthropic-budget-effort`), `efforts`, `defaultLevel`, `effortMap`, `supportsDisplay`, `requiresEffort` (legacy `levels` / `minLevel`+`maxLevel` accepted and normalized). |
| `input` | `("text" \| "image")[]`. |
| `imageInputDecoder` | `stb` — local STB decoder; OMP converts WebP before dispatch for backends that cannot accept it. |
| `tokenizer` | Force an embedded local tokenizer: `claude-v3`, `claude-v47`, `claude-v5`, `claude-v5-sonnet`, `qwen3`, `deepseek-v3`, `kimi-k2`, `glm5`. |
| `supportsTools` | Boolean. |
| `cost` | `input`, `output`, `cacheRead`, `cacheWrite` (definition requires all four). |
| `premiumMultiplier` | Premium-tier price multiplier. |
| `contextWindow`, `maxTokens` | Token limits. |
| `omitMaxOutputTokens`, `preferWebsockets` | Transport toggles. |
| `headers` | Per-model headers. |
| `compat` | Compatibility flags. |
| `contextPromotionTarget` | Model to promote to on context overflow. |
| `compactionModel` | Model used to summarize/compact this model's session. |
| `remoteCompaction` | Provider-native compaction: `enabled`, `api`, `endpoint`, `model`, `v2StreamingEnabled`, `v2Endpoint`, `streamingEndpoint`. |

### `compat` flags

OpenAI-family: `supportsStore`, `supportsDeveloperRole`, `supportsMultipleSystemMessages`, `supportsReasoningEffort`, `reasoningEffortMap`, `maxTokensField`, `supportsUsageInStreaming`, `requiresToolResultName`, `requiresMistralToolIds`, `requiresAssistantAfterToolResult`, `requiresThinkingAsText`, `reasoningContentField`, `requiresReasoningContentForToolCalls`, `allowsSyntheticReasoningContentForToolCalls`, `requiresAssistantContentForToolCalls`, `supportsToolChoice`, `supportsForcedToolChoice`, `disableReasoningOnForcedToolChoice`, `thinkingFormat` (`openai`/`openrouter`/`zai`/`qwen`/`qwen-chat-template`), `openRouterRouting`, `vercelGatewayRouting`, `extraBody`, `cacheControlFormat`, `supportsStrictMode`, `toolStrictMode`, `streamIdleTimeoutMs`, `streamMarkupHealingPattern`, `supportsLongPromptCacheRetention`, `supportsReasoningParams`, `supportsReasoningSummary`, `alwaysSendMaxTokens`, `strictResponsesPairing`, `supportsImageDetailOriginal`, `whenThinking` (conditional overrides).

Anthropic-messages flags (same slot): `supportsContextManagement`, `supportsEagerToolInputStreaming`, `allowAnthropicHeaderOverrides`, `requiresToolResultId`, `replayUnsignedThinking`.

Bedrock: `promptCacheMode` (`none`/`automatic`/`explicit`), `supportsLongPromptCacheRetention`, `promptCacheMinimumTokens`, `promptCacheMaximumCheckpoints`.

---

## Dynamic Runtime Discovery

OMP can query a local or proxy endpoint to discover models at startup.

| Discovery type | Description | Required provider settings |
| --- | --- | --- |
| `ollama` | Queries `/api/tags` and `/api/show` | `baseUrl` (e.g. `http://127.0.0.1:11434`), `api: openai-responses` |
| `llama.cpp` | Queries native llama.cpp model endpoints | `baseUrl` (e.g. `http://127.0.0.1:8080`), `api: openai-responses` |
| `lm-studio` | Queries OpenAI-compatible `GET /v1/models` | `baseUrl` (e.g. `http://127.0.0.1:1234/v1`), `api: openai-completions` |
| `litellm` | Queries LiteLLM management info and `/v1/models` | `baseUrl` (e.g. `http://localhost:4000/v1`), `api: openai-completions` |
| `openai-models-list` | Plain OpenAI-compatible `GET /v1/models` | `baseUrl`; set `injectV1: false` for gateways rooted at a versioned path |
| `proxy` | Auto-detects Anthropic vs OpenAI behind a shared proxy | `baseUrl` |

Discovery options: `timeoutMs` (positive finite; per-provider HTTP probe timeout) and `injectV1` (default `true`, `openai-models-list` only).

---

## Command-Resolved Secrets (`!command`)

Prefix any `apiKey` or header value with `!` to execute a shell command on demand instead of hardcoding plaintext:

```yaml
providers:
  openai-custom:
    baseUrl: https://api.openai.com/v1
    apiKey: "!op read op://Personal/OpenAI/credential"

  anthropic-custom:
    baseUrl: https://api.anthropic.com/v1
    apiKey: "!bw get password anthropic-api-key"
    headers:
      X-Custom-Auth: "!security find-generic-password -s my-secret -w"
```

OMP executes the command (10-second timeout), trims whitespace, and caches the result for the process lifetime.

---

## Validation Rules

**Full custom provider** (`models` non-empty):
- `baseUrl` required.
- `apiKey` required unless `auth: none`.
- `api` required at provider level or on each model.

**Partial override provider** (only `modelOverrides` / provider settings): `apiKey` is not required; overrides patch bundled models by id.

**Model definition**: `id` non-empty; `name`, `baseUrl`, `contextPromotionTarget`, `compactionModel` must be non-empty when present; `cost` requires all four sub-fields.

Errors surface via `ModelRegistry.getError()` in the UI.

---

## Concrete Recipes

### 1. Local Ollama
```yaml
providers:
  ollama:
    baseUrl: http://127.0.0.1:11434
    api: openai-responses
    auth: none
    discovery:
      type: ollama
      timeoutMs: 5000
```

### 2. Local vLLM / SGLang / LM Studio
```yaml
providers:
  vllm:
    baseUrl: http://127.0.0.1:8000/v1
    apiKey: "none"
    api: openai-completions
    models:
      - id: meta-llama/Llama-3.3-70B-Instruct
        name: Llama 3.3 70B
        contextWindow: 131072
        maxTokens: 8192
        reasoning: false
        input: [text]
      - id: Qwen/Qwen2.5-Coder-32B-Instruct
        name: Qwen 2.5 Coder 32B
        contextWindow: 65536
        maxTokens: 8192
        reasoning: false
        input: [text]
```

### 3. OpenRouter Custom Routing
```yaml
providers:
  openrouter:
    baseUrl: https://openrouter.ai/api/v1
    apiKey: "!echo $OPENROUTER_API_KEY"
    api: openai-completions
    models:
      - id: anthropic/claude-3.7-sonnet
        name: Claude 3.7 Sonnet (OpenRouter)
        contextWindow: 200000
        maxTokens: 8192
        compat:
          openRouterRouting:
            order: ["Anthropic", "Together"]
```

### 4. Overriding Bundled Model Metadata (`modelOverrides`)
```yaml
providers:
  anthropic:
    modelOverrides:
      claude-sonnet-4-5:
        contextWindow: 250000
        supportsTools: true
        compactionModel: anthropic/claude-haiku-4-5
```

### 5. Auth-Gateway `pi-native` Transport
```yaml
providers:
  gateway:
    baseUrl: http://127.0.0.1:8787
    apiKey: "!echo $OMP_GATEWAY_BEARER"
    transport: pi-native
```
