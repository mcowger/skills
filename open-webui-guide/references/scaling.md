# Scaling and Production deployment

## Contents
1. [Architecture for scaling](#architecture-for-scaling)
2. [Production deployment checklist](#production-deployment-checklist)
3. [Database](#database)
4. [Redis](#redis)
5. [WebSocket cluster](#websocket-cluster)
6. [Horizontal scaling](#horizontal-scaling)
7. [Docker Compose for production](#docker-compose-for-production)
8. [Kubernetes](#kubernetes)
9. [Reverse proxy](#reverse-proxy)
10. [Monitoring](#monitoring)
11. [Backups](#backups)
12. [Common issues when scaling](#common-issues)

---

## Architecture for scaling

### Single server (up to ~50 users)

```
[Nginx] → [Open WebUI] → [SQLite] + [Ollama]
```

Suitable for small teams. SQLite with WAL, Ollama on the same machine.

### Medium scale (50-500 users)

```
[Nginx/Traefik]
    ├── [Open WebUI #1] ──┐
    ├── [Open WebUI #2] ──┤── [PostgreSQL]
    └── [Open WebUI #3] ──┘        │
              │                     │
         [Redis] ←──────────────────┘
              │
    [Ollama / vLLM cluster]
```

### High load (500+ users)

```
[CDN/WAF]
    │
[Load Balancer (sticky sessions)]
    ├── [Open WebUI Pod #1-N] ──── [PostgreSQL Primary]
    │         │                         │
    │    [Redis Cluster/Sentinel]  [PostgreSQL Replica]
    │
    ├── [Ollama Pool]
    ├── [vLLM / TGI Pool]
    └── [Pipeline Servers]
```

---

## Production deployment checklist

### Required

- [ ] `WEBUI_SECRET_KEY` — stable random key (don't change it after deployment!)
- [ ] PostgreSQL instead of SQLite
- [ ] Redis for WebSocket and token revocation
- [ ] HTTPS (via reverse proxy)
- [ ] `WEBUI_AUTH_COOKIE_SECURE=true`
- [ ] `DEFAULT_USER_ROLE=pending` (manual user approval)
- [ ] DB backups

### Recommended

- [ ] `ENABLE_AUDIT_LOGS_FILE=true`
- [ ] OpenTelemetry for monitoring
- [ ] Rate limiting at the reverse proxy level
- [ ] Restrict `CORS_ALLOW_ORIGIN` to a specific domain
- [ ] Tune `DATABASE_POOL_SIZE` for the load
- [ ] Resource monitoring (CPU, RAM, GPU)

---

## Database

### PostgreSQL (recommended for production)

```env
DATABASE_URL=postgresql://openwebui:<your-password>@db.example.com:5432/openwebui
DATABASE_POOL_SIZE=50
DATABASE_POOL_MAX_OVERFLOW=20
DATABASE_POOL_TIMEOUT=30
```

### PostgreSQL tuning

```sql
-- postgresql.conf
shared_buffers = 256MB           -- 25% RAM
effective_cache_size = 768MB     -- 75% RAM
work_mem = 16MB
maintenance_work_mem = 128MB
max_connections = 200
```

### Connection Pooling

For high load, use PgBouncer:

```ini
# pgbouncer.ini
[databases]
openwebui = host=localhost dbname=openwebui

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
```

### Migrations

Open WebUI uses Alembic for migrations. They run automatically when the application starts. When upgrading the version:

1. Back up the DB
2. Update the container/code
3. Migrations will be applied on the first startup

---

## Redis

### Standalone

```env
REDIS_URL=redis://redis.example.com:6379/0
REDIS_KEY_PREFIX=owui:
```

### Sentinel (High Availability)

```env
REDIS_SENTINEL_HOSTS=sentinel1:26379,sentinel2:26379,sentinel3:26379
REDIS_SENTINEL_PORT=26379
```

### Cluster

```env
REDIS_URL=redis://redis-cluster:6379
REDIS_CLUSTER=true
```

### What Redis is needed for

| Function | Without Redis | With Redis |
|---------|-----------|---------|
| JWT revocation | Doesn't work | Works |
| WebSocket cluster | Doesn't work | Works |
| Caching | Local | Distributed |
| Pub/Sub | Doesn't work | Works |

**Conclusion**: Redis is required for any cluster of 2+ Open WebUI instances.

---

## WebSocket cluster

Without proper WebSocket configuration, Open WebUI cannot be scaled horizontally — realtime messages will be lost.

```env
ENABLE_WEBSOCKET_SUPPORT=true
WEBSOCKET_MANAGER=redis
WEBSOCKET_REDIS_URL=redis://redis:6379/1
```

This makes Socket.IO use Redis as a message broker, so messages are delivered to all connected clients, regardless of which instance they're connected to.

### Sticky Sessions

Socket.IO requires sticky sessions when using the long-polling fallback. Configure the load balancer:

**Nginx:**
```nginx
upstream openwebui {
    ip_hash;  # sticky sessions
    server webui1:8080;
    server webui2:8080;
}
```

**Traefik:**
```yaml
services:
  openwebui:
    loadBalancer:
      sticky:
        cookie:
          name: owui_session
```

---

## Horizontal scaling

### What scales

- **Open WebUI (stateless)** — horizontally, N instances behind a load balancer
- **PostgreSQL** — vertically or via read replicas
- **Redis** — Sentinel or Cluster
- **Ollama** — each instance = 1 GPU, scales horizontally
- **Pipeline servers** — independently

### What does NOT scale automatically

- **SQLite** — one file, one writer. Replace with PostgreSQL
- **Local WebSocket manager** — replace with Redis
- **File storage** — needs shared storage (NFS, S3 + a layer)

### File storage in a cluster

Open WebUI stores uploaded files in `DATA_DIR/uploads/`. With multiple instances, all of them must see the same files:

- **NFS**: mount `DATA_DIR` over NFS
- **S3-compatible**: requires custom integration or a volume plugin
- **Single Volume (Docker)**: if all containers are on the same host

---

## Docker Compose for production

```yaml
version: '3.8'

services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 2G
    environment:
      - WEBUI_SECRET_KEY=${SECRET_KEY}
      - DATABASE_URL=postgresql://owui:${DB_PASS}@db:5432/openwebui
      - REDIS_URL=redis://redis:6379/0
      - WEBSOCKET_MANAGER=redis
      - WEBSOCKET_REDIS_URL=redis://redis:6379/1
      - OLLAMA_BASE_URL=http://ollama:11434
      - ENABLE_WEBSOCKET_SUPPORT=true
      - WEBUI_AUTH_COOKIE_SECURE=true
      - DEFAULT_USER_ROLE=pending
    volumes:
      - shared-data:/app/backend/data
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: openwebui
      POSTGRES_USER: owui
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data

  ollama:
    image: ollama/ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    volumes:
      - ollama-models:/root/.ollama

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - openwebui

volumes:
  shared-data:
  pgdata:
  redisdata:
  ollama-models:
```

---

## Kubernetes

### Core resources

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-webui
spec:
  replicas: 3
  selector:
    matchLabels:
      app: open-webui
  template:
    spec:
      containers:
        - name: open-webui
          image: ghcr.io/open-webui/open-webui:main
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: owui-secrets
                  key: database-url
            - name: REDIS_URL
              value: redis://redis-svc:6379/0
            - name: WEBSOCKET_MANAGER
              value: redis
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
          volumeMounts:
            - name: data
              mountPath: /app/backend/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: owui-data
```

### Health check

Open WebUI provides a `/health` endpoint for status checks.

---

## Reverse proxy

### Nginx

```nginx
server {
    listen 443 ssl http2;
    server_name chat.example.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # WebSocket support
    location /ws {
        proxy_pass http://openwebui;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Socket.IO
    location /socket.io {
        proxy_pass http://openwebui;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    location / {
        proxy_pass http://openwebui;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # For file uploads
        client_max_body_size 100M;
    }

    upstream openwebui {
        ip_hash;
        server webui1:8080;
        server webui2:8080;
    }
}
```

Critical points:
- **WebSocket**: be sure to proxy `/socket.io` with the Upgrade headers
- **Timeout**: `proxy_read_timeout 86400` for long streaming responses
- **Body size**: `client_max_body_size` for file uploads

---

## Monitoring

### OpenTelemetry

```env
ENABLE_OTEL=true
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_SERVICE_NAME=open-webui
```

Exports:
- **Traces** — requests through all layers
- **Metrics** — counters, histograms
- **Logs** — structured logs

Connects to Jaeger, Grafana Tempo, Datadog, etc.

### Audit logs

```env
ENABLE_AUDIT_LOGS_FILE=true
ENABLE_AUDIT_STDOUT=true
AUDIT_LOG_LEVEL=REQUEST        # NONE | METADATA | REQUEST | REQUEST_RESPONSE
```

- `METADATA` — who, when, which endpoint
- `REQUEST` — + request body
- `REQUEST_RESPONSE` — + response body (caution: large volume!)

---

## Backups

### PostgreSQL

```bash
# Daily backup
pg_dump -U owui openwebui | gzip > backup_$(date +%Y%m%d).sql.gz

# Restore
gunzip < backup_20240101.sql.gz | psql -U owui openwebui
```

### SQLite

```bash
# Backup (safe, with WAL)
sqlite3 data/webui.db ".backup data/webui_backup.db"
```

### Files

Don't forget to back up:
- `DATA_DIR/uploads/` — uploaded files
- `DATA_DIR/vector_db/` — vector DB (if using Chroma)
- The `.env` file or secrets

---

## Common issues

### WebSocket doesn't work in a cluster
**Cause**: local WebSocket manager, messages don't reach other instances.
**Solution**: `WEBSOCKET_MANAGER=redis` + `WEBSOCKET_REDIS_URL=...`

### Sessions "reset" on restart
**Cause**: `WEBUI_SECRET_KEY` isn't set — a random one is generated on every startup.
**Solution**: set a fixed `WEBUI_SECRET_KEY`.

### Files not visible on another instance
**Cause**: `DATA_DIR` isn't shared between containers.
**Solution**: NFS, shared volume, or S3.

### Slow embeddings
**Cause**: CPU inference of sentence-transformers.
**Solution**: `SENTENCE_TRANSFORMERS_BACKEND=cuda` + GPU, or an external embedding service.

### OOM when uploading large files for RAG
**Cause**: document too large, chunking consumes memory.
**Solution**: increase the container's memory limits, decrease `RAG_CHUNK_SIZE`.
