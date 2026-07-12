# Hidden features and non-obvious functionality

## Contents
1. [Easter Eggs](#easter-eggs)
2. [Advanced admin settings](#advanced-admin-settings)
3. [Non-obvious environment variables](#non-obvious-environment-variables)
4. [Hidden API endpoints](#hidden-api-endpoints)
5. [Advanced function patterns](#advanced-function-patterns)
6. [Useful tricks](#useful-tricks)
7. [Enterprise features](#enterprise-features)
8. [Pitfalls](#pitfalls)

---

## Easter Eggs

```env
ENABLE_EASTER_EGGS=true   # enabled by default
```

Open WebUI contains hidden UI elements activated by this variable. Disable it in enterprise deployments if you want a strict interface.

---

## Advanced admin settings

### Auto-creating an admin on first startup

```env
WEBUI_ADMIN_EMAIL=admin@company.com
WEBUI_ADMIN_NAME=System Admin
ENABLE_INITIAL_ADMIN_SIGNUP=true
# After bootstrap, remove these variables and disable registration: ENABLE_SIGNUP=false
```

After the first startup, these variables can be removed — the account has already been created.

### Programmatic registration control

```env
ENABLE_SIGNUP=false              # close registration entirely
DEFAULT_USER_ROLE=pending        # or: registration open, but requires approval
```

### OAuth Token Exchange

```env
ENABLE_OAUTH_TOKEN_EXCHANGE=true
```

Allows exchanging the OAuth provider's tokens for Open WebUI's internal JWT. Useful for integrations where an external service has already authenticated the user.

### Forwarding User Info

```env
ENABLE_FORWARD_USER_INFO_HEADERS=false
# Only enable for trusted internal providers and with an approved PII policy.
```

Passes user information (email, name, role) in headers of requests to LLM providers. Useful for:
- Audit logging on the provider side
- Per-user rate limiting
- Response personalization

Headers: `X-OpenWebUI-User-Email`, `X-OpenWebUI-User-Name`, `X-OpenWebUI-User-Id`.

---

## Non-obvious environment variables

### OFFLINE_MODE

```env
OFFLINE_MODE=true
```

Completely disables all outgoing HTTP requests. Open WebUI will not:
- Check for updates
- Download models
- Reach out to external APIs

Ideal for air-gapped environments (closed networks, military/government systems).

### Licensing

```env
LICENSE_KEY=<your-license-key>
LICENSE_BLOB_PATH=/path/to/license.blob
```

Enterprise license. Unlocks additional capabilities (SSO, extended auditing, etc.).

### TRUSTED_SIGNATURE_KEY

```env
TRUSTED_SIGNATURE_KEY=<your-hmac-key>
```

HMAC key for signing webhooks. Allows the recipient to verify that a webhook was actually sent by Open WebUI.

### Multiple providers

```env
OLLAMA_BASE_URLS="http://ollama1:11434;http://ollama2:11434"
OPENAI_API_BASE_URLS="https://api.openai.com/v1;http://vllm:8000/v1;http://litellm:4000"
OPENAI_API_KEYS="<key-1>;none;<key-2>"
```

You can connect multiple Ollama servers and OpenAI-compatible APIs simultaneously. Keys and URLs are matched by position, separated by `;`.

### Database Session Sharing

```env
DATABASE_ENABLE_SESSION_SHARING=true
```

Reuses DB sessions across requests. Can improve performance, but requires care with async operations.

---

## Hidden API endpoints

### /health

```bash
curl http://localhost:8080/health
```

Health check. Returns `{"status": true}` if the service is alive. Use for readiness/liveness probes.

### /api/config

```bash
curl http://localhost:8080/api/config
```

Public configuration (no secrets). Shows which features are enabled, version, etc.

### /api/v1/auths/api-key — with expiry

```bash
curl -X POST http://localhost:8080/api/v1/auths/api-key \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"expires_in": "30d"}'
```

You can set the API key's lifetime.

### Sync functions

```bash
curl -X POST http://localhost:8080/api/v1/functions/sync \
  -H "Authorization: Bearer $TOKEN" \
  -d '[{...}, {...}]'
```

Bulk function synchronization — useful for CI/CD.

### Secure function deployment

Only load functions from vetted local code and deploy via a trusted CI/CD pipeline:

```text
POST /api/v1/functions/create
POST /api/v1/functions/sync
```

Don't deploy executable function code from arbitrary public URLs.

---

## Advanced function patterns

### Filter chain with priorities

```python
# Filter 1: priority=0 (first) — moderation
# Filter 2: priority=10 — context enrichment
# Filter 3: priority=99 (last) — logging

class Filter:
    class Valves(BaseModel):
        priority: int = Field(default=0)
```

### Function as authentication middleware

```python
class Filter:
    async def inlet(self, body, __user__=None):
        if __user__ and __user__.get("role") != "admin":
            # Restrict access to a specific model
            if body.get("model") == "gpt-4":
                raise Exception("You do not have access to GPT-4")
        return body
```

### Streaming pipe

```python
import httpx

class Pipe:
    async def pipe(self, body, __user__=None):
        async def stream():
            async with httpx.AsyncClient() as client:
                async with client.stream("POST", self.valves.api_url, json=body) as resp:
                    async for chunk in resp.aiter_text():
                        yield chunk
        return stream()
```

### Event emitter for progress

```python
class Action:
    async def action(self, body, __user__=None, __event_emitter__=None):
        total = 10
        for i in range(total):
            await __event_emitter__({
                "type": "status",
                "data": {
                    "description": f"Processing {i+1}/{total}...",
                    "done": False
                }
            })
            # ... work ...

        await __event_emitter__({
            "type": "status",
            "data": {"description": "Done!", "done": True}
        })
```

---

## Useful tricks

### 1. Custom models (Model Cards)

Through the admin panel you can create a "virtual model" — a wrapper over a real model with:
- A custom system prompt
- Preset parameters (temperature, top_p)
- Attached knowledge bases
- A custom name and description

Users only see this "model," without knowing the implementation details.

### 2. Prompt templates

Saved prompts are invoked via `/command` in the chat input field. Create via:
- UI: Workspace → Prompts → Create
- API: `POST /api/v1/prompts/create`

### 3. Channels for team collaboration

Channels are analogous to Slack channels, but with AI. Multiple users see the same chat and can interact with the model together.

### 4. User Presence

```
Profile settings → Status
```

Users can set a status (online/offline/busy) and a text message with an emoji. Visible to other users.

### 5. Markdown + LaTeX in chat

Open WebUI renders Markdown and LaTeX ($...$, $$...$$) in messages. Models can use this to format responses.

### 6. Keyboard shortcuts

- `Enter` — send message
- `Shift+Enter` — new line
- `Ctrl+Shift+;` — voice input
- `/` — commands/prompts

### 7. Chat export/import

Chats can be exported to JSON via the UI or API. This allows:
- Moving chats between instances
- Backing up individual chats
- Sharing chats with colleagues

### 8. Files in chat

Dragging files into the chat automatically uses them as context. Supported:
- Text files
- PDF
- Images (if the model is multimodal)

---

## Enterprise features

### SCIM Provisioning

Automatic user sync from Azure AD / Okta:
```env
ENABLE_SCIM=true
SCIM_AUTH_TOKEN=<your-token>
```

### Audit logs

```env
ENABLE_AUDIT_LOGS_FILE=true
AUDIT_LOG_LEVEL=REQUEST_RESPONSE
```

A complete log: who, when, what was requested and what was returned. For compliance (SOC2, ISO 27001).

### OpenTelemetry

```env
ENABLE_OTEL=true
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
```

Distributed tracing, metrics, logs — connects to Grafana, Datadog, Jaeger.

### Webhook Signing

```env
TRUSTED_SIGNATURE_KEY=<your-hmac-key>
```

All outgoing webhooks are signed with HMAC — the recipient can verify authenticity.

---

## Pitfalls

### 1. WEBUI_SECRET_KEY — the most common mistake

Without this variable, the JWT secret is generated randomly on every startup. Consequences:
- All sessions are invalidated on restart
- In a cluster, each instance generates its own secret — tokens are not valid across instances

**Solution**: always set `WEBUI_SECRET_KEY` in production.

### 2. SQLite is not for clustering

SQLite supports only a single writer. Two Open WebUI instances sharing a single SQLite file will guarantee data corruption.

### 3. bcrypt and long passwords

bcrypt truncates the password at 72 bytes. With Unicode (Cyrillic characters = 2 bytes) that's even fewer characters. A 40-character Cyrillic password = 80 bytes → gets truncated.

### 4. Filter functions and infinite loops

If an outlet filter calls another model, the request will pass through all filters again. Use a flag in the body to prevent this:

```python
async def outlet(self, body, __user__=None):
    if body.get("_processed_by_my_filter"):
        return body
    body["_processed_by_my_filter"] = True
    # ... processing ...
    return body
```

### 5. Order of OPENAI_API_BASE_URLS

URLs and keys are matched by position:
```env
OPENAI_API_BASE_URLS="http://a:8000;http://b:8000"
OPENAI_API_KEYS="key-a;key-b"
```

If there are three URLs but only two keys, the third URL will be left without a key. Always check the counts.

### 6. Docker and host.docker.internal

- **Docker Desktop** (Windows/macOS): `host.docker.internal` works out of the box
- **Linux**: you need to add `--add-host=host.docker.internal:host-gateway` or use the host's IP

### 7. Embedding model and language

The default `all-MiniLM-L6-v2` is optimized for English. For Russian text, RAG will perform poorly. Replace it with `intfloat/multilingual-e5-large` or a similar model.

### 8. WebSocket behind Cloudflare

By default, Cloudflare can close WebSocket connections after 100 seconds. Configure:
- Cloudflare Dashboard → Network → WebSockets → Enable
- Increase the timeout in Origin Rules settings

### 9. Data loss when updating Docker

If data isn't in a volume — it only lives inside the container:
```bash
# CORRECT: named volume
docker run -v open-webui-data:/app/backend/data ...

# INCORRECT: data inside the container
docker run ...  # after docker rm — data is lost
```

### 10. CORS in production

```env
CORS_ALLOW_ORIGIN=*   # ← DO NOT DO THIS in production!
CORS_ALLOW_ORIGIN=https://chat.example.com  # ← correct
```
