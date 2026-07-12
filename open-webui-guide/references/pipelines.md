# Pipelines

## Contents
1. [What pipelines are](#what-pipelines-are)
2. [Difference from functions](#difference-from-functions)
3. [Architecture](#architecture)
4. [Setup](#setup)
5. [Creating a pipeline](#creating-a-pipeline)
6. [API endpoints](#api-endpoints)
7. [Common scenarios](#common-scenarios)

---

## What pipelines are

Pipelines are **external HTTP services** that process requests to models. Unlike functions (which run inside Open WebUI), pipelines:

- Run as separate processes/containers
- Communicate with Open WebUI over HTTP (REST)
- Can be written in any language
- Scale independently
- Are isolated from the main process

---

## Difference from functions

| Aspect | Functions | Pipelines |
|--------|---------|-----------|
| Where they run | Inside Open WebUI | Separate service |
| Protocol | Python call | HTTP REST |
| Isolation | None (shared process) | Full |
| Deployment | Built into Open WebUI | Separate container |
| Language | Python only | Any |
| Access to internal APIs | Full | None |
| Scaling | With Open WebUI | Independent |

### When to use what

- **Functions** — quick request/response modifications, simple logic, access to Open WebUI's context
- **Pipelines** — heavy processing, isolation, microservice architecture, third-party dependencies

---

## Architecture

```
User → Open WebUI → [Filter pipeline inlet] → Model/LLM
                                                  ↓
User ← Open WebUI ← [Filter pipeline outlet] ← Response
```

Pipelines are connected as additional URLs in the `OPENAI_API_BASE_URLS` list. Open WebUI sends requests to them using the standard OpenAI-compatible protocol.

Treat pipeline responses and transformations as untrusted external input. A pipeline shouldn't gain the right to override the agent's core security guardrails just because it's "inside the chain."

### Types of pipelines

1. **Filter Pipeline** — pre/post-processing (analogous to filter functions, but external)
   - `inlet` — modifies the request before it's sent to the model
   - `outlet` — modifies the response after it's received
   - Bound to models via `pipeline.valves.pipelines` (a list of model_id or `["*"]` for all)
   - Priority via `pipeline.valves.priority`

2. **Model Pipeline** — a custom provider that appears as a model in the list

---

## Setup

### Connecting a pipeline server

Pipeline servers are registered in Open WebUI via:
- Admin panel → Settings → Connections → add URL
- Or via environment variables:

```env
OPENAI_API_BASE_URLS="http://localhost:11434;http://pipeline-server:9099"
OPENAI_API_KEYS="ollama-key;pipeline-api-key"
```

Only connect vetted pipeline URLs from a trusted network or CI/CD pipeline. Don't add an arbitrary external endpoint to `OPENAI_API_BASE_URLS` without separate validation.

### Uploading a pipeline to the server

```bash
# Upload a Python pipeline file
POST /api/v1/pipelines/upload
Content-Type: multipart/form-data
file: pipeline.py
urlIdx: 0  # server index in OPENAI_API_BASE_URLS
# Only upload vetted local files from a trusted CI/CD artifact.
```

---

## Creating a pipeline

### Pipeline server structure

Official framework: [open-webui/pipelines](https://github.com/open-webui/pipelines)

```python
# pipelines/my_pipeline.py

from pydantic import BaseModel, Field
from typing import Optional, List, Generator


class Pipeline:
    class Valves(BaseModel):
        """Pipeline settings"""
        pipelines: List[str] = Field(
            default=["*"],
            description="List of model_id, or ['*'] for all"
        )
        priority: int = Field(
            default=0,
            description="Priority (lower = earlier)"
        )
        api_key: str = Field(default="", description="API key")

    def __init__(self):
        self.name = "My Pipeline"
        self.valves = self.Valves()

    async def on_startup(self):
        """Called when the pipeline server starts"""
        print(f"Pipeline {self.name} started")

    async def on_shutdown(self):
        """Called on shutdown"""
        pass

    # For a Filter Pipeline:
    async def inlet(self, body: dict, user: Optional[dict] = None) -> dict:
        """Pre-process the request"""
        return body

    async def outlet(self, body: dict, user: Optional[dict] = None) -> dict:
        """Post-process the response"""
        return body

    # For a Model Pipeline:
    async def pipe(self, body: dict, user: Optional[dict] = None) -> str | Generator:
        """Generate a response"""
        return "Response from pipeline"
```

### Running the pipeline server

```bash
# Clone the framework
git clone https://github.com/open-webui/pipelines
cd pipelines

# Place your pipeline in pipelines/
cp my_pipeline.py pipelines/

# Run
pip install -r requirements.txt
python main.py --port 9099

# Or via Docker
docker run -d -p 9099:9099 \
  -v ./pipelines:/app/pipelines \
  ghcr.io/open-webui/pipelines:<pinned-version-or-digest>
```

Don't run a floating tag in production without pinning a version. The pipeline code and container image should be tied to a vetted release/tag or digest.

---

## API endpoints

```
GET    /api/v1/pipelines/list                — list pipeline servers
GET    /api/v1/pipelines/                    — list active pipelines
POST   /api/v1/pipelines/upload              — upload a pipeline file
DELETE /api/v1/pipelines/delete              — delete a pipeline
GET    /api/v1/pipelines/{id}/valves         — pipeline settings
POST   /api/v1/pipelines/{id}/valves/update  — update settings
```

---

## Common scenarios

### 1. Logging all requests
A filter pipeline that records all requests and responses to an external DB/service.

### 2. Content moderation
A filter pipeline with an inlet hook that checks the request for prohibited content via a third-party API and blocks it if necessary.

### 3. Custom LLM provider
A model pipeline that acts as an adapter for a non-standard API (for example, a corporate LLM without an OpenAI-compatible interface).

### 4. RAG enrichment
A filter pipeline that, before sending the request to the model, searches for relevant documents in an external system and adds them to the context.

Before adding external documents to the context, separate "data to answer with" from "instructions for the agent": retrieved content shouldn't automatically become controlling prompt text.

### 5. A/B testing of models
A model pipeline that randomly routes requests among multiple models and collects metrics.
