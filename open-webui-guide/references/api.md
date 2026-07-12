# API Reference

## Contents
1. [API authentication](#api-authentication)
2. [Base URL](#base-url)
3. [Routers by category](#routers-by-category)
4. [OpenAI-compatible API](#openai-compatible-api)
5. [Request examples](#request-examples)

---

## API authentication

All API requests (except `/api/v1/auths/signin` and `/api/v1/auths/signup`) require authorization:

```bash
# Via JWT token (obtained at login)
Authorization: Bearer eyJhbGciOiJI...

# Via API key
Authorization: Bearer sk-xxxxxxxx...
```

---

## Base URL

```
http://localhost:8080/api/v1/
```

In a Docker container, the default port is `8080`, often mapped to `3000`.

---

## Routers by category

### Authentication (`/api/v1/auths/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| POST | `/signin` | Sign in with password | Public |
| POST | `/signup` | Sign up | Public |
| POST | `/ldap` | Sign in via LDAP | Public |
| GET | `/oauth/{provider}/authorize` | OAuth authorization | Public |
| POST | `/logout` | Sign out (revoke token) | Authorized |
| POST | `/api-key` | Create API key | Authorized |
| DELETE | `/api-key` | Delete API key | Authorized |
| PATCH | `/profile` | Update profile | Authorized |
| PATCH | `/password/update` | Change password | Authorized |

### Users (`/api/v1/users/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List users | Admin |
| GET | `/{id}` | User profile | Admin |
| POST | `/{id}/update` | Update user | Admin |
| DELETE | `/{id}` | Delete user | Admin |
| GET | `/me` | Current user | Authorized |
| POST | `/{id}/role` | Change role | Admin |

### Chats (`/api/v1/chats/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List chats | Authorized |
| POST | `/new` | Create chat | Authorized |
| GET | `/{id}` | Get chat | Owner |
| POST | `/{id}` | Update chat | Owner |
| DELETE | `/{id}` | Delete chat | Owner |
| GET | `/all` | All chats (admin) | Admin |
| POST | `/{id}/archive` | Archive | Owner |
| POST | `/{id}/share` | Share chat | Owner |
| POST | `/{id}/clone` | Clone chat | Authorized |
| GET | `/tags` | Chat tags | Authorized |

### Models (`/api/v1/models/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List models | Authorized |
| POST | `/create` | Create model | Admin |
| GET | `/{id}` | Model info | Authorized |
| POST | `/{id}/update` | Update model | Admin |
| DELETE | `/{id}/delete` | Delete model | Admin |

### Functions (`/api/v1/functions/`)

See `functions.md` for the full list.

### Pipelines (`/api/v1/pipelines/`)

See `pipelines.md` for the full list.

### Knowledge bases (`/api/v1/knowledge/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List knowledge bases | Authorized |
| POST | `/` | Create knowledge base | Authorized |
| GET | `/{id}` | Knowledge base info | Authorized |
| POST | `/{id}/update` | Update | Owner/Admin |
| DELETE | `/{id}/delete` | Delete | Owner/Admin |
| POST | `/{id}/files` | Add file | Owner/Admin |
| GET | `/{id}/files` | Knowledge base files | Authorized |
| DELETE | `/{id}/files/{file_id}` | Delete file | Owner/Admin |

### Files (`/api/v1/files/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| POST | `/` | Upload file | Authorized |
| GET | `/` | List files | Authorized |
| GET | `/{id}` | File metadata | Authorized |
| GET | `/{id}/content` | File content | Authorized |
| DELETE | `/{id}` | Delete file | Owner/Admin |

### Prompts (`/api/v1/prompts/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List prompts | Authorized |
| POST | `/create` | Create prompt | Authorized |
| GET | `/{command}` | Get prompt | Authorized |
| POST | `/{command}/update` | Update prompt | Owner/Admin |
| DELETE | `/{command}/delete` | Delete prompt | Owner/Admin |

### Tools (`/api/v1/tools/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List tools | Authorized |
| POST | `/create` | Create tool | Admin |
| GET | `/id/{id}` | Info | Authorized |
| POST | `/id/{id}/update` | Update | Admin |
| DELETE | `/id/{id}/delete` | Delete | Admin |

### Channels (`/api/v1/channels/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List channels | Authorized |
| POST | `/create` | Create channel | Admin |
| GET | `/{id}` | Info | Member |
| POST | `/{id}/update` | Update | Admin |
| DELETE | `/{id}/delete` | Delete | Admin |
| GET | `/{id}/messages` | Channel messages | Member |
| POST | `/{id}/messages/post` | Send message | Member |

### Groups (`/api/v1/groups/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | List groups | Authorized |
| POST | `/create` | Create group | Admin |
| GET | `/{id}` | Info | Authorized |
| POST | `/{id}/update` | Update | Admin |
| DELETE | `/{id}/delete` | Delete | Admin |

### Configuration (`/api/v1/configs/`)

| Method | Path | Description | Access |
|-------|------|----------|--------|
| GET | `/` | Current configuration | Admin |
| POST | `/update` | Update configuration | Admin |

### Other routers

| Router | Path | Purpose |
|--------|------|------------|
| Audio | `/api/v1/audio/` | TTS/STT (speech synthesis and recognition) |
| Images | `/api/v1/images/` | Image generation (DALL-E, Stable Diffusion) |
| Retrieval | `/api/v1/retrieval/` | Web search, document processing |
| Memories | `/api/v1/memories/` | User memories |
| Notes | `/api/v1/notes/` | Notes |
| Folders | `/api/v1/folders/` | Organizational folders |
| Tags | `/api/v1/tags/` | Tags |
| Evaluations | `/api/v1/evaluations/` | Model evaluation |
| Analytics | `/api/v1/analytics/` | Usage analytics |
| SCIM | `/api/v1/scim/` | SCIM 2.0 provisioning |
| Terminals | `/api/v1/terminals/` | Code execution |

---

## OpenAI-compatible API

Open WebUI provides an OpenAI-compatible API, allowing it to be used as a drop-in replacement:

```bash
# Chat Completions
curl -X POST http://localhost:8080/api/chat/completions \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": false
  }'

# List Models
curl http://localhost:8080/api/models \
  -H "Authorization: Bearer <your-api-key>"
```

This allows connecting Open WebUI to any application that can work with the OpenAI API (LangChain, AutoGen, etc.).

---

## Request examples

### Create a chat and send a message

```python
import requests

BASE = "http://localhost:8080/api/v1"
TOKEN = "<your-jwt-token>"
headers = {"Authorization": f"Bearer {TOKEN}"}

# Create a chat
chat = requests.post(f"{BASE}/chats/new", json={
    "chat": {"title": "Test chat", "messages": []}
}, headers=headers).json()

# Send a message via the OpenAI-compatible API
response = requests.post("http://localhost:8080/api/chat/completions", json={
    "model": "llama3.1",
    "messages": [{"role": "user", "content": "What is Python?"}],
    "stream": False
}, headers=headers).json()
```

### Upload a file to a knowledge base

```python
# 1. Upload file
with open("document.pdf", "rb") as f:
    file_resp = requests.post(f"{BASE}/files/",
        files={"file": f},
        headers=headers
    ).json()

# 2. Create knowledge base
kb = requests.post(f"{BASE}/knowledge/", json={
    "name": "Documentation",
    "description": "Project documentation"
}, headers=headers).json()

# 3. Add file to knowledge base
requests.post(f"{BASE}/knowledge/{kb['id']}/files", json={
    "file_id": file_resp["id"]
}, headers=headers)
```
