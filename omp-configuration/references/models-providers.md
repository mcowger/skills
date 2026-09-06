# OMP Models & Providers Reference (`models.yml`)

This reference covers custom model definitions, local LLM integrations, OpenAI/Anthropic-compatible endpoints, discovery mechanisms, and secret resolution in `~/.omp/agent/models.yml`.

## File Location & Syntax

- Primary Path: `~/.omp/agent/models.yml` (or `.yaml`)
- Top-level structure MUST contain a `providers` mapping.

```yaml
providers:
  <provider-id>:
    baseUrl: <url>
    apiKey: <key-or-command>
    api: <api-type>
    headers:
      <header-name>: <header-value>
    discovery:
      type: <discovery-type>
    models:
      - id: <model-id>
        name: <display-name>
        contextWindow: <tokens>
        maxTokens: <tokens>
        reasoning: <boolean>
        input: [text, image]
```

---

## Supported `api` Protocol Types

| API Protocol | Description | Common Backends |
| --- | --- | --- |
| `openai-completions` | Standard OpenAI Chat Completions (`/v1/chat/completions`) | vLLM, LM Studio, Ollama, LiteLLM, Groq |
| `openai-responses` | OpenAI Responses API (`/v1/responses`) | OpenAI official, Ollama native, LiteLLM Responses |
| `openai-codex-responses` | OpenAI Codex streaming protocol | Codex-optimized endpoints |
| `anthropic-messages` | Anthropic Messages API (`/v1/messages`) | Anthropic official, Bedrock, Anthropic proxies |
| `google-vertex` | Google Cloud Vertex AI protocol | Vertex AI Gemini models |
| `google-generative-ai` | Google AI Studio Gemini API | Gemini Developer API |
| `bedrock-converse-stream` | AWS Bedrock Converse Stream | AWS Bedrock Claude & Llama models |

---

## Dynamic Runtime Discovery

OMP can automatically query a local or proxy endpoint to discover available models at startup.

| Discovery Type | Description | Required Provider Settings |
| --- | --- | --- |
| `ollama` | Queries `/api/tags` and `/api/show` | `baseUrl` (e.g. `http://127.0.0.1:11434`), `api: openai-responses` |
| `llama.cpp` | Queries native llama.cpp model endpoints | `baseUrl` (e.g. `http://127.0.0.1:8080`), `api: openai-responses` |
| `lm-studio` | Queries OpenAI-compatible `GET /v1/models` | `baseUrl` (e.g. `http://127.0.0.1:1234/v1`), `api: openai-completions` |
| `litellm` | Queries LiteLLM management info and `/v1/models` | `baseUrl` (e.g. `http://localhost:4000/v1`), `api: openai-completions` |
| `proxy` | Auto-detects Anthropic vs OpenAI endpoints behind a shared proxy | `baseUrl` (e.g. `http://proxy.local/v1`) |

---

## Command-Resolved Secrets (`!command`)

To avoid hardcoding plaintext API keys or tokens in `models.yml`, prefix any `apiKey` or header value with `!` to execute a shell command on demand:

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

## Concrete Recipes

### 1. Local Ollama Integration
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

### 4. Overriding Built-in Model Metadata (`modelOverrides`)
```yaml
providers:
  anthropic:
    modelOverrides:
      claude-sonnet-4-5:
        contextWindow: 250000
        supportsTools: true
```
