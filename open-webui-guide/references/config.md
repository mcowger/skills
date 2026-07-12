# Configuration and environment variables

## Contents
1. [How configuration works](#how-configuration-works)
2. [Critical variables](#critical-variables)
3. [Authentication](#authentication)
4. [Database](#database)
5. [Redis](#redis)
6. [LLM providers](#llm-providers)
7. [RAG and embeddings](#rag-and-embeddings)
8. [WebSocket](#websocket)
9. [Audio (TTS/STT)](#audio)
10. [Images](#images)
11. [Logging and monitoring](#logging-and-monitoring)
12. [Security](#security)
13. [Other settings](#other-settings)

---

## How configuration works

Open WebUI reads its configuration from two sources:
1. **Environment variables** (`env.py`) — read at startup, don't change without a restart
2. **DB configuration** (`config.py`) — dynamic, changes via the admin panel or API

Priority: environment variables > DB. Some settings are only available via env, others via both mechanisms.

Key files:
- `backend/open_webui/env.py` — environment variables
- `backend/open_webui/config.py` — configuration logic

---

## Critical variables

These variables **must** be set in production:

```env
# JWT secret — without it, sessions reset on every restart!
WEBUI_SECRET_KEY=your-secure-random-string-here

# Database — SQLite by default, use PostgreSQL for production
DATABASE_URL=postgresql://user:pass@localhost:5432/openwebui

# Ollama URL (if using local models)
OLLAMA_BASE_URL=http://localhost:11434

# Redis (for scaling and token revocation)
REDIS_URL=redis://localhost:6379/0
```

---

## Authentication

```env
# Core
WEBUI_AUTH=true                          # enable authentication
WEBUI_SECRET_KEY=                        # JWT secret (REQUIRED for production!)
JWT_EXPIRES_IN=24h                       # token lifetime

# Registration
ENABLE_SIGNUP=true                       # allow registration
ENABLE_PASSWORD_AUTH=true                # allow password login
ENABLE_INITIAL_ADMIN_SIGNUP=true         # first user = admin
DEFAULT_USER_ROLE=pending                # role for new users

# Auto-create admin
WEBUI_ADMIN_EMAIL=admin@example.com
WEBUI_ADMIN_PASSWORD=
WEBUI_ADMIN_NAME=Admin

# Passwords
ENABLE_PASSWORD_VALIDATION=false
PASSWORD_VALIDATION_REGEX_PATTERN=

# Cookies
WEBUI_SESSION_COOKIE_SAME_SITE=lax       # lax | strict | none
WEBUI_AUTH_COOKIE_SECURE=false            # true for HTTPS
WEBUI_AUTH_COOKIE_HTTP_ONLY=true

# OAuth
ENABLE_OAUTH_SIGNUP=false
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=false
OAUTH_MAX_SESSIONS_PER_USER=10
ENABLE_OAUTH_TOKEN_EXCHANGE=false
ENABLE_OAUTH_ID_TOKEN_COOKIE=false

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_OAUTH_SCOPE=openid email profile
GOOGLE_REDIRECT_URI=

# Microsoft OAuth
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
MICROSOFT_CLIENT_TENANT_ID=

# GitHub OAuth
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=

# Generic OIDC
OAUTH_CLIENT_ID=
OAUTH_CLIENT_SECRET=
OAUTH_PROVIDER_NAME=
OAUTH_OPENID_CONFIG_URL=
OAUTH_SCOPES=openid email profile
OAUTH_REDIRECT_URI=

# LDAP
ENABLE_LDAP=false
LDAP_SERVER_HOST=
LDAP_SERVER_PORT=389
LDAP_USE_TLS=true
LDAP_BIND_DN=
LDAP_BIND_PASSWORD=
LDAP_SEARCH_BASE=
LDAP_SEARCH_FILTERS=
LDAP_ATTRIBUTE_FOR_MAIL=mail
LDAP_ATTRIBUTE_FOR_USERNAME=uid

# Trusted Headers
WEBUI_AUTH_TRUSTED_EMAIL_HEADER=
WEBUI_AUTH_TRUSTED_NAME_HEADER=
WEBUI_AUTH_TRUSTED_GROUPS_HEADER=

# SCIM
ENABLE_SCIM=false
SCIM_AUTH_TOKEN=
```

---

## Database

```env
# Connection (choose one format)
DATABASE_URL=sqlite:///data/webui.db          # SQLite (default)
DATABASE_URL=postgresql://user:pass@host/db   # PostgreSQL
DATABASE_URL=mysql://user:pass@host/db        # MySQL

# Or by parts
DATABASE_TYPE=postgresql
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=openwebui
DATABASE_USER=user
DATABASE_PASSWORD=pass

# Connection pool
DATABASE_POOL_SIZE=50                    # pool size (default: depends on DB)
DATABASE_POOL_MAX_OVERFLOW=20            # extra connections beyond the pool
DATABASE_POOL_TIMEOUT=30                 # connection wait timeout (sec)

# SQLite-specific
DATABASE_ENABLE_SQLITE_WAL=true          # Write-Ahead Logging (recommended)

# General
DATABASE_ENABLE_SESSION_SHARING=false    # session reuse
```

### Recommendations

- **Development**: SQLite with WAL — simple and sufficient
- **Production < 50 users**: SQLite with WAL, file on SSD
- **Production 50+ users**: PostgreSQL
- **High load**: PostgreSQL + connection pooler (PgBouncer)

---

## Redis

```env
# Main connection
REDIS_URL=redis://localhost:6379/0
REDIS_KEY_PREFIX=open-webui:            # key prefix (for shared Redis)

# Cluster
REDIS_CLUSTER=false

# Sentinel (HA)
REDIS_SENTINEL_HOSTS=host1:26379,host2:26379
REDIS_SENTINEL_PORT=26379
```

Redis is used for:
- JWT token revocation (JTI blacklist)
- WebSocket manager (for clustering)
- Caching
- Pub/Sub (realtime notifications)

---

## LLM providers

```env
# Ollama
OLLAMA_BASE_URL=http://localhost:11434       # Ollama server URL
OLLAMA_BASE_URLS=url1;url2                   # multiple Ollama servers

# OpenAI-compatible API
OPENAI_API_BASE_URL=https://api.openai.com/v1
OPENAI_API_BASE_URLS=url1;url2;url3          # multiple providers
OPENAI_API_KEY=<your-api-key>
OPENAI_API_KEYS=<key-1>;<key-2>;<key-3>       # keys for each URL

# Default models
DEFAULT_MODELS=llama3.1                      # default model for new chats

# Access control
BYPASS_MODEL_ACCESS_CONTROL=false            # bypass model access control
ENABLE_FORWARD_USER_INFO_HEADERS=false       # pass user info in headers
```

---

## RAG and embeddings

```env
# Embedding model
RAG_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
SENTENCE_TRANSFORMERS_BACKEND=torch          # torch | cuda | onnx

# Vector DB
VECTOR_DB=chroma                             # chroma | milvus | weaviate | qdrant | opensearch | pgvector
CHROMA_DATA_DIR=./data/vector_db
# Or for external ones:
MILVUS_URI=http://localhost:19530
WEAVIATE_URL=http://localhost:8083
QDRANT_URI=http://localhost:6333

# Chunking parameters
RAG_CHUNK_SIZE=1500                          # chunk size (characters)
RAG_CHUNK_OVERLAP=100                        # chunk overlap

# Reranking
RAG_RERANKING_MODEL=                         # reranking model (optional)

# Timeouts
RAG_EMBEDDING_TIMEOUT=60                     # embedding generation timeout (sec)
```

---

## WebSocket

```env
ENABLE_WEBSOCKET_SUPPORT=true               # enable WebSocket
WEBSOCKET_MANAGER=                           # empty = local, "redis" = Redis adapter
WEBSOCKET_REDIS_URL=redis://localhost:6379/1
WEBSOCKET_REDIS_CLUSTER=false
WEBSOCKET_SERVER_PING_TIMEOUT=20             # ping timeout (sec)
WEBSOCKET_SERVER_PING_INTERVAL=25            # ping interval (sec)
```

---

## Audio

```env
# Text-to-Speech
TTS_ENGINE=                                  # openai | elevenlabs | azure
TTS_MODEL=tts-1
TTS_VOICE=alloy
OPENAI_API_KEY=                              # if TTS via OpenAI

# Speech-to-Text
STT_ENGINE=                                  # openai | whisper
WHISPER_MODEL=base                           # tiny | base | small | medium | large
```

---

## Images

```env
IMAGE_GENERATION_ENGINE=                     # openai | comfyui | automatic1111
IMAGES_OPENAI_API_BASE_URL=https://api.openai.com/v1
IMAGES_OPENAI_API_KEY=
IMAGE_GENERATION_MODEL=dall-e-3
IMAGE_SIZE=1024x1024
```

---

## Logging and monitoring

```env
# Logging
GLOBAL_LOG_LEVEL=INFO                        # DEBUG | INFO | WARNING | ERROR
LOG_FORMAT=                                  # empty = text, "json" = JSON

# Audit
ENABLE_AUDIT_LOGS_FILE=false                 # write audit logs to file
ENABLE_AUDIT_STDOUT=false                    # output audit to stdout
AUDIT_LOG_LEVEL=METADATA                     # NONE | METADATA | REQUEST | REQUEST_RESPONSE

# OpenTelemetry
ENABLE_OTEL=false                            # enable OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=open-webui
```

---

## Security

```env
# Webhook signing
TRUSTED_SIGNATURE_KEY=                       # HMAC key for signing webhooks

# Licensing (Enterprise)
LICENSE_KEY=
LICENSE_BLOB_PATH=

# CORS
CORS_ALLOW_ORIGIN=https://chat.example.com   # allowed origins (do NOT use * in production!)
```

---

## Other settings

```env
# General
PORT=8080                                    # server port
HOST=0.0.0.0                                 # bind address
OFFLINE_MODE=false                           # disable external requests
DATA_DIR=./data                              # data directory

# Features
ENABLE_EASTER_EGGS=true                      # Easter eggs in the UI
ENABLE_VERSION_UPDATE_CHECK=true             # check for updates
ENABLE_REALTIME_CHAT_SAVE=true               # auto-save chats
ENABLE_CHAT_RESPONSE_BASE64_IMAGE_URL_CONVERSION=false  # base64 image conversion
```
