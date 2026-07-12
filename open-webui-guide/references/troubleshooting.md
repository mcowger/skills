# Debugging and troubleshooting

## Contents
1. [Diagnostics](#diagnostics)
2. [Startup issues](#startup-issues)
3. [Authorization issues](#authorization-issues)
4. [Model issues](#model-issues)
5. [RAG/Knowledge issues](#rag-issues)
6. [WebSocket issues](#websocket-issues)
7. [Performance issues](#performance-issues)
8. [Docker issues](#docker-issues)
9. [Logging](#logging)

---

## Diagnostics

### First steps for any issue

1. **Check the logs**:
   ```bash
   docker logs open-webui --tail 100
   docker compose logs -f openwebui
   ```

2. **Check health**:
   ```bash
   curl http://localhost:8080/health
   ```

3. **Enable DEBUG logging**:
   ```env
   GLOBAL_LOG_LEVEL=DEBUG
   ```

4. **Check environment variables**:
   ```bash
   docker exec open-webui env | sort
   ```

---

## Startup issues

### Container crashes immediately after starting

**Port already in use**:
```bash
lsof -i :8080
# or
netstat -tlnp | grep 8080
```

**No permissions on the data directory**:
```bash
ls -la /path/to/data
chmod -R 777 /path/to/data  # debugging only!
```

**DB migration error**:
```bash
docker logs open-webui 2>&1 | grep -i "migration\|alembic\|error"
```
Solution: back up the DB, then recreate it, or manually run `alembic upgrade head`.

---

## Authorization issues

### "Invalid credentials"

1. Password: bcrypt truncates anything past 72 bytes — long passwords may not work
2. Forgot the admin password — set a new one via env:
   ```env
   WEBUI_ADMIN_EMAIL=new@email.com
   WEBUI_ADMIN_PASSWORD=<one-time-strong-secret>
   ```

### Sessions reset after restart

`WEBUI_SECRET_KEY` isn't set — a random one is generated on every startup, invalidating old JWTs:
```env
WEBUI_SECRET_KEY=my-permanent-secret-key
```

### OAuth — redirect_uri mismatch

1. The URL in env must exactly match what's configured at the provider
2. Check the protocol (`https` vs `http`) and trailing slash
3. Behind a reverse proxy: the URL must be the external one, not the internal one

### OAuth — account isn't created

- `ENABLE_OAUTH_SIGNUP=true` — allow registration via OAuth
- `DEFAULT_USER_ROLE=pending` — user is created, but waits for approval

### API key doesn't work

- Format: `Authorization: Bearer sk-xxxxx`
- Check `expires_at` — the key may have expired
- Check the role of the user who owns the key

---

## Model issues

### Ollama models not visible

1. Check the URL from within the container:
   ```bash
   docker exec open-webui curl http://ollama:11434/api/tags
   ```
2. If Ollama is on the host: `OLLAMA_BASE_URL=http://host.docker.internal:11434`
3. Is Ollama only listening on localhost? Run: `OLLAMA_HOST=0.0.0.0 ollama serve`

### Streaming cuts off

1. Reverse proxy — increase timeouts:
   ```nginx
   proxy_read_timeout 300;
   proxy_send_timeout 300;
   proxy_buffering off;
   ```
2. Cloudflare: enable WebSocket support
3. AWS ALB: increase the idle timeout

### Slow generation

- `ollama ps` — check that the model is on the GPU
- `nvidia-smi` — is there free VRAM
- Low VRAM — use a quantized model (Q4_K_M)

---

## RAG issues

### File isn't indexed

1. Check the format: PDF, DOCX, TXT, MD, CSV are supported
2. Scanned PDFs need OCR
3. Check the logs: `docker logs open-webui 2>&1 | grep -i "embed\|chunk\|rag"`

### Irrelevant results

1. Embedding model: for Russian text — `multilingual-e5-large` or `intfloat/multilingual-e5-base`
2. Decrease `RAG_CHUNK_SIZE` (800–1000) and increase `RAG_CHUNK_OVERLAP` (200)
3. Enable reranking: `RAG_RERANKING_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2`

### OOM during indexing

- Increase the container's RAM
- Use a lightweight embedding model (`all-MiniLM-L6-v2`)

---

## WebSocket issues

### Chat doesn't update in real time

1. `ENABLE_WEBSOCKET_SUPPORT=true`
2. Reverse proxy — headers are required:
   ```nginx
   proxy_http_version 1.1;
   proxy_set_header Upgrade $http_upgrade;
   proxy_set_header Connection "upgrade";
   ```
3. DevTools → Network → WS — check the handshake

### Frequent reconnects

- Increase `WEBSOCKET_SERVER_PING_TIMEOUT=60`
- Nginx: `proxy_read_timeout 86400;`

### Cluster: events don't arrive

- Is `WEBSOCKET_MANAGER=redis` configured?
- Is Redis reachable from all instances?
- Sticky sessions on the LB?

---

## Performance issues

### High CPU

- Embeddings load the CPU → switch to GPU or an external service
- SQLite under load → migrate to PostgreSQL

### High memory

- Embedding model in RAM → use a smaller one or offload it
- Memory leak → Docker restart policy (`unless-stopped`)

### Slow UI

- Too many chats → archive old ones
- Large chats → split into new ones

---

## Docker issues

### No GPU

```bash
# Check the runtime
docker run --rm --gpus all nvidia/cuda:12.1.0-base-ubuntu22.04 nvidia-smi
# Nothing? → install nvidia-container-toolkit
```

### Data lost after recreation

Be sure to mount a volume:
```bash
docker run -v open-webui-data:/app/backend/data ...
```

### Can't connect to localhost

From inside the container, `localhost` = the container itself. Use:
- `host.docker.internal` (Docker Desktop)
- `172.17.0.1` (Linux bridge)
- Docker network + service name

---

## Logging

```env
GLOBAL_LOG_LEVEL=DEBUG     # DEBUG | INFO | WARNING | ERROR
LOG_FORMAT=json            # for ELK/Loki

# Audit logs
ENABLE_AUDIT_LOGS_FILE=true
ENABLE_AUDIT_STDOUT=true
AUDIT_LOG_LEVEL=REQUEST    # NONE | METADATA | REQUEST | REQUEST_RESPONSE
```

### Useful grep patterns

```bash
docker logs open-webui 2>&1 | grep -i "error\|exception\|traceback"
docker logs open-webui 2>&1 | grep -i "auth\|login\|token"
docker logs open-webui 2>&1 | grep -i "ollama\|openai\|model"
docker logs open-webui 2>&1 | grep -i "socket\|websocket"
```
