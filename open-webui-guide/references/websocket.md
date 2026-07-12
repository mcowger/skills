# WebSocket and realtime

## Contents
1. [Architecture](#architecture)
2. [Setup](#setup)
3. [Events](#events)
4. [Cluster mode](#cluster-mode)
5. [Collaborative editing (Y.js)](#collaborative-editing)
6. [Debugging](#debugging)

---

## Architecture

Open WebUI uses **Socket.IO** on top of WebSocket for realtime features:
- Realtime chat updates
- Streaming model responses
- User presence (online/offline)
- Collaborative editing (Y.js CRDT)
- Notifications

Key file: `backend/open_webui/socket/main.py`

### Stack

```
Client (SvelteKit) ←→ Socket.IO ←→ FastAPI (asyncio)
                                        ↕
                                    [Redis adapter] (for clustering)
```

---

## Setup

### Basic (single instance)

```env
ENABLE_WEBSOCKET_SUPPORT=true
# WEBSOCKET_MANAGER=           # empty = local manager
```

### Cluster (multiple instances)

```env
ENABLE_WEBSOCKET_SUPPORT=true
WEBSOCKET_MANAGER=redis
WEBSOCKET_REDIS_URL=redis://redis:6379/1
WEBSOCKET_REDIS_CLUSTER=false
```

### Timeouts

```env
WEBSOCKET_SERVER_PING_TIMEOUT=20     # seconds before disconnect if no pong
WEBSOCKET_SERVER_PING_INTERVAL=25    # seconds between pings
```

---

## Events

### Connect/disconnect

On connection, the client sends a JWT token. The server:
1. Validates the token
2. Registers the user in a room by user_id
3. Updates the presence status

### Main events

| Event | Direction | Description |
|---------|-------------|----------|
| `connect` | Client → Server | Connection with JWT |
| `disconnect` | Client → Server | Disconnection |
| `chat-update` | Server → Client | Chat update |
| `message` | Both | Message in a channel |
| `presence` | Server → Client | User status |
| `typing` | Client → Server | Typing indicator |

### Room-based broadcast

Each user is placed in a "room" by their user_id. This allows sending events to a specific user:

```python
# Server side
await sio.emit("chat-update", data, room=user_id)
```

---

## Cluster mode

With multiple Open WebUI instances behind a load balancer:

1. **Redis adapter**: Socket.IO uses Redis Pub/Sub to synchronize events between instances
2. **Sticky sessions**: Socket.IO's long-polling fallback requires the client to always land on the same instance

Without a Redis adapter: clients connected to instance A won't receive events sent from instance B.

---

## Collaborative editing

Open WebUI supports collaborative editing via **Y.js** — a CRDT (Conflict-free Replicated Data Types) library.

This allows multiple users to simultaneously edit:
- Notes
- Prompts
- Other text data

Y.js synchronizes over the same Socket.IO channel.

---

## Debugging

### WebSocket doesn't connect

1. Check that `ENABLE_WEBSOCKET_SUPPORT=true`
2. Check the reverse proxy — the `Upgrade` and `Connection` headers are required
3. In the browser: DevTools → Network → WS — check the handshake status

### Messages don't arrive in a cluster

1. Check `WEBSOCKET_MANAGER=redis`
2. Check Redis availability: `redis-cli ping`
3. Check that all instances use the same Redis

### Frequent reconnects

1. Increase `WEBSOCKET_SERVER_PING_TIMEOUT`
2. Check that the reverse proxy isn't killing idle connections (nginx: `proxy_read_timeout 86400`)
3. Check network stability
