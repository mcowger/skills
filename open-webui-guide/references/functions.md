# Functions

## Contents
1. [What functions are](#what-functions-are)
2. [Types of functions](#types-of-functions)
3. [Function structure](#function-structure)
4. [Valves (settings)](#valves)
5. [Lifecycle](#lifecycle)
6. [API endpoints](#api-endpoints)
7. [Pitfalls](#pitfalls)

---

## What functions are

Functions are Python modules that execute **inside** the Open WebUI process. They allow you to modify requests/responses, create custom model providers, and add actions to the UI.

Unlike pipelines (external HTTP services), functions:
- Run in the same process as Open WebUI
- Have access to internal APIs and context
- Don't require a separate deployment
- Are loaded dynamically (hot-reload)

---

## Types of functions

### 1. Filter

Intercepts model requests and responses. Two hooks:

- **inlet** — called BEFORE the request is sent to the model. Can modify the prompt, add context, validate/reject the request.
- **outlet** — called AFTER the response is received. Can modify the response, add metadata, log.

Filters run in priority order (the `priority` field). Can be enabled/disabled globally (`is_global`) or bound to specific models.

### 2. Pipe (custom provider)

A custom model provider. A pipe function generates the response itself — Open WebUI shows it as a separate "model" in the list. Use it for:
- Wrapping non-standard APIs
- Aggregating multiple models
- Custom generation logic

### 3. Action

An action button in the chat UI. When clicked by the user, Python code is invoked with access to the message context. Examples:
- "Translate to Russian"
- "Create a Jira ticket"
- "Save to notes"

---

## Function structure

### Minimal template (Filter)

```python
"""
title: My Filter
author: Your Name
version: 0.1.0
"""

from pydantic import BaseModel, Field
from typing import Optional


class Filter:
    class Valves(BaseModel):
        """Settings available to the administrator"""
        enabled: bool = Field(default=True, description="Enable the filter")
        max_tokens: int = Field(default=4096, description="Maximum tokens")

    class UserValves(BaseModel):
        """Settings available to each user"""
        language: str = Field(default="en", description="Response language")

    def __init__(self):
        self.valves = self.Valves()

    async def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        """Called before the request is sent to the model"""
        # body contains: messages, model, stream, etc.
        if self.valves.enabled:
            # Example: add a system message
            body["messages"].insert(0, {
                "role": "system",
                "content": f"Respond in language: {self.valves.language}"
            })
        return body

    async def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        """Called after the response is received"""
        # body contains the model's response
        return body
```

### Minimal template (Pipe)

```python
"""
title: Custom Model
author: Your Name
version: 0.1.0
"""

from pydantic import BaseModel, Field
from typing import Optional, Generator


class Pipe:
    class Valves(BaseModel):
        api_key: str = Field(default="", description="External service API key")
        api_url: str = Field(default="https://api.example.com", description="API URL")

    def __init__(self):
        self.valves = self.Valves()
        # Defines the model name(s) in the list. Can be a list for multiple models:
        # self.pipes = [{"id": "model-a", "name": "Model A"}, {"id": "model-b", "name": "Model B"}]

    def pipes(self) -> list[dict]:
        """Returns the list of models provided by this pipe"""
        return [{"id": "my-custom-model", "name": "My Custom Model"}]

    async def pipe(self, body: dict, __user__: Optional[dict] = None) -> str | Generator:
        """Generates a response"""
        messages = body.get("messages", [])
        # Implement the call to the external API
        # For streaming, return a generator; for a regular response, return a string
        return "Response from the custom model"
```

### Minimal template (Action)

```python
"""
title: Translate to Russian
author: Your Name
version: 0.1.0
"""

from pydantic import BaseModel


class Action:
    class Valves(BaseModel):
        pass

    def __init__(self):
        self.valves = self.Valves()

    async def action(
        self,
        body: dict,
        __user__: Optional[dict] = None,
        __event_emitter__=None,
    ) -> Optional[dict]:
        """Called when the action button is clicked"""
        # body contains the current message context
        message = body.get("messages", [])[-1]

        if __event_emitter__:
            await __event_emitter__(
                {
                    "type": "status",
                    "data": {"description": "Translating...", "done": False},
                }
            )

        # ... perform the action ...

        if __event_emitter__:
            await __event_emitter__(
                {
                    "type": "status",
                    "data": {"description": "Done!", "done": True},
                }
            )
```

---

## Valves

Valves are the configuration mechanism for functions, based on Pydantic models.

### Two levels

1. **Valves** (class-level) — configured by the administrator via UI or API
2. **UserValves** — configured individually by each user

### Supported field types

```python
class Valves(BaseModel):
    # Simple types
    enabled: bool = Field(default=True)
    temperature: float = Field(default=0.7, ge=0, le=2)
    max_tokens: int = Field(default=4096)
    api_key: str = Field(default="")

    # Lists
    allowed_models: list[str] = Field(default=["gpt-4", "gpt-3.5-turbo"])

    # Enum via Literal
    mode: Literal["fast", "accurate"] = Field(default="fast")
```

### Dynamic options (dropdown)

Valves support dynamic dropdown lists via a method on the function's class.

---

## Special parameters

The `inlet`, `outlet`, `pipe`, and `action` methods have access to special double-underscore parameters:

| Parameter | Description |
|----------|----------|
| `__user__` | Dict with user data: `{id, email, name, role}` |
| `__event_emitter__` | Function to send events to the UI (status, progress) |
| `__event_call__` | Function to request input from the user |
| `__id__` | Function ID |
| `__model__` | Info about the current model |
| `__chat_id__` | Current chat ID |
| `__message_id__` | Current message ID |

---

## Lifecycle

### Loading
1. Function code is stored in the DB (the `function` table)
2. When activated, Open WebUI dynamically imports the Python module
3. Import auto-replacement is performed for safety
4. An instance of the class is created

### Execution (Filter)
```
User request
  → Filter 1 inlet (priority=0)
  → Filter 2 inlet (priority=1)
  → Model generates response
  → Filter 2 outlet
  → Filter 1 outlet
  → Response to user
```

### Global functions
`is_global=True` — applies to ALL models. Otherwise, it's bound to specific models via the UI.

---

## API endpoints

```
GET    /api/v1/functions/              — list functions
POST   /api/v1/functions/create        — create function
POST   /api/v1/functions/sync          — bulk sync
GET    /api/v1/functions/id/{id}       — get function
POST   /api/v1/functions/id/{id}/update     — update
DELETE /api/v1/functions/id/{id}/delete      — delete
POST   /api/v1/functions/id/{id}/toggle     — enable/disable
POST   /api/v1/functions/id/{id}/toggle/global  — global mode
GET    /api/v1/functions/id/{id}/valves          — current valves
POST   /api/v1/functions/id/{id}/valves/update   — update valves
GET    /api/v1/functions/id/{id}/valves/user     — user valves
POST   /api/v1/functions/id/{id}/valves/user/update — update user valves
```

---

## Pitfalls

### 1. Function ID — lowercase + underscore only
```
my_filter     ✓
My-Filter     ✗
my filter     ✗
```

### 2. Module resource limits
Function code executes in the main process — a poorly written function can:
- Block the event loop (sync I/O without async)
- Leak memory
- Crash and affect the entire service

Always use `async` for I/O operations.

### 3. Filter order matters
Filters with a lower `priority` run first. If one filter modifies `messages`, the next filter will receive the already-modified version.

### 4. Auto-import replacement
Open WebUI automatically replaces some imports for safety. If your import doesn't work, this may be the cause.

### 5. Hot reload
After a function's code is updated via the UI, it's reloaded automatically. But if the function stores state in `__init__`, that state will be reset.
