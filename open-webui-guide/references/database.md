# Database

## Contents
1. [Technologies](#technologies)
2. [Tables and models](#tables-and-models)
3. [Migrations](#migrations)
4. [Working with the DB directly](#working-with-the-db-directly)

---

## Technologies

- **ORM**: SQLAlchemy 2.x (async)
- **Migrations**: Alembic (automatic at startup)
- **Supported DBMSs**: SQLite, PostgreSQL, MySQL

Key files:
- `backend/open_webui/internal/db.py` — connection, sessions, engine
- `backend/open_webui/models/` — ORM models (23+ files)
- `backend/open_webui/migrations/` — Alembic migrations

---

## Tables and models

### Users and authentication

**user** — main user table
```
id              UUID, PK
email           VARCHAR, unique
username        VARCHAR
name            VARCHAR
role            VARCHAR (admin | user | pending)
profile_image_url  VARCHAR
bio             TEXT
gender          VARCHAR
DOB             DATE
timezone        VARCHAR
presence_state  VARCHAR (online | offline | away)
status_emoji    VARCHAR
status_message  VARCHAR
settings        JSON — user settings
oauth           JSON — OAuth metadata
scim            JSON — SCIM metadata
last_active_at  TIMESTAMP
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**auth** — password authentication
```
id              UUID, PK (FK → user.id)
email           VARCHAR
password        VARCHAR (bcrypt hash)
active          BOOLEAN
```

**api_key** — API keys
```
id              UUID, PK
user_id         UUID, FK → user.id
key             VARCHAR (sk-xxx)
data            JSON (metadata)
expires_at      TIMESTAMP
last_used_at    TIMESTAMP
created_at      TIMESTAMP
```

**oauth_session** — OAuth sessions
```
id              UUID, PK
user_id         UUID, FK → user.id
provider        VARCHAR
access_token    TEXT
refresh_token   TEXT
expires_at      TIMESTAMP
created_at      TIMESTAMP
```

### Chats and messages

**chat** — chat sessions
```
id              UUID, PK
user_id         UUID, FK → user.id
title           VARCHAR
chat_template   JSON
documents       JSON (attached documents)
models          JSON (models used)
messages        JSON (legacy — message array)
pinned          BOOLEAN
archived        BOOLEAN
folder_id       UUID, FK → folder.id
share_id        VARCHAR (for shared links)
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**chat_message** — individual messages (newer model)
```
id              UUID, PK
chat_id         UUID, FK → chat.id
message_id      VARCHAR
user_id         UUID
content         TEXT
role            VARCHAR (user | assistant | system)
files           JSON
citations       JSON
created_at      TIMESTAMP
```

### Content and knowledge

**knowledge** — knowledge bases
```
id              UUID, PK
user_id         UUID, FK → user.id
name            VARCHAR
description     TEXT
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**knowledge_file** — link between files and a knowledge base
```
id              UUID, PK
knowledge_id    UUID, FK → knowledge.id
file_id         UUID, FK → file.id
user_id         UUID
```

**file** — uploaded files
```
id              UUID, PK
user_id         UUID, FK → user.id
filename        VARCHAR
data            JSON (file path, size, MIME type)
meta            JSON (additional metadata)
created_at      TIMESTAMP
```

**prompt** — saved prompts
```
id              UUID, PK (= command)
user_id         UUID, FK → user.id
command         VARCHAR (unique)
title           VARCHAR
content         TEXT
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**note** — notes
```
id              UUID, PK
user_id         UUID, FK → user.id
title           VARCHAR
content         TEXT
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**memory** — user memories
```
id              UUID, PK
user_id         UUID, FK → user.id
content         TEXT
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### Functions and tools

**function** — user-defined functions
```
id              VARCHAR, PK (lowercase + underscore)
user_id         UUID, FK → user.id
name            VARCHAR
type            VARCHAR (filter | pipe | action)
content         TEXT (Python code)
meta            JSON (description, manifest)
valves          JSON (admin settings)
is_active       BOOLEAN
is_global       BOOLEAN
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**tool** — tools for function calling
```
id              VARCHAR, PK
user_id         UUID, FK → user.id
name            VARCHAR
content         TEXT (Python code)
specs           JSON (OpenAPI specification)
meta            JSON
valves          JSON
is_active       BOOLEAN
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### Models

**model** — model definitions
```
id              VARCHAR, PK (model ID)
user_id         UUID, FK → user.id
base_model_id   VARCHAR (base model ID)
name            VARCHAR
params          JSON (temperature, top_p, etc.)
meta            JSON (description, capabilities)
access_control  JSON (who has access)
is_active       BOOLEAN
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### Organization and access

**folder** — folders
```
id              UUID, PK
user_id         UUID, FK → user.id
name            VARCHAR
parent_id       UUID, FK → folder.id (nesting)
is_expanded     BOOLEAN
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**group** — user groups
```
id              UUID, PK
name            VARCHAR
description     TEXT
permissions     JSON
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**access_grant** — granular access permissions
```
id              UUID, PK
resource_type   VARCHAR (knowledge | model | chat | ...)
resource_id     UUID
target_type     VARCHAR (user | group)
target_id       UUID
access_level    VARCHAR (read | write | admin)
created_at      TIMESTAMP
```

**tag** — tags
```
id              UUID, PK
user_id         UUID
name            VARCHAR
data            JSON
meta            JSON
```

### Collaboration

**channel** — channels
```
id              UUID, PK
name            VARCHAR
description     TEXT
access_control  JSON
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**message** — channel messages
```
id              UUID, PK
channel_id      UUID, FK → channel.id
user_id         UUID, FK → user.id
content         TEXT
data            JSON
meta            JSON
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

### Analytics

**feedback** — feedback on responses
```
id              UUID, PK
user_id         UUID
type            VARCHAR (thumbs_up | thumbs_down)
data            JSON
meta            JSON
created_at      TIMESTAMP
```

---

## Migrations

### Automatic migrations

Open WebUI runs Alembic migrations automatically on every startup. Migration files are located in `backend/open_webui/migrations/versions/`.

### Creating a migration (for developers)

```bash
cd backend
alembic revision --autogenerate -m "description of changes"
alembic upgrade head
```

### Rolling back a migration

```bash
cd backend
alembic downgrade -1    # roll back the last one
alembic downgrade <rev> # roll back to a specific revision
```

---

## Working with the DB directly

### SQLite

```bash
sqlite3 data/webui.db
.tables          # list of tables
.schema user     # table structure
SELECT * FROM user LIMIT 5;
```

### PostgreSQL

```bash
psql -U owui openwebui
\dt              # list of tables
\d user          # table structure
SELECT id, email, role FROM "user" LIMIT 5;
```

Note: in PostgreSQL, `user` is a reserved word, so quotes `"user"` are required.
