---
name: open-terminal-guide
description: >-
  Comprehensive Open Terminal reference: a self-hosted terminal REST API for
  AI agents. Use this skill for any questions about Open Terminal: API
  endpoints, deployment (Docker, bare metal), configuration, integration with
  Open WebUI, architecture, debugging, and writing client code that interacts
  with Open Terminal. Also use it when working with the open-terminal codebase:
  project structure, runner.py, main.py, the MCP server, and Jupyter notebooks.
  Triggers: open-terminal, open terminal, remote terminal API, /execute, /files,
  sandbox API, terminal sessions, Open WebUI terminal.
---

# Open Terminal — Complete Reference

Open Terminal is a lightweight, self-hosted REST API that provides AI agents and automation tools with a dedicated environment for executing commands, managing files, and running code. It is part of the [Open WebUI](https://github.com/open-webui/open-webui) ecosystem.

## Two Operating Modes

| Mode | Description | Security |
|------|-------------|----------|
| **Docker** | An isolated container with Python, Node.js, git, ffmpeg, LaTeX, data science libraries, and more preinstalled | Sandboxed and safe for AI agents |
| **Bare metal** | Run `pip install open-terminal` and launch it directly on the host | Commands run with the user's privileges—no isolation |

## Quick Start

```bash
# Set the key in advance through a secret manager or env file
export OPEN_TERMINAL_API_KEY='<random-secret-from-secret-store>'
export OPEN_TERMINAL_IMAGE='ghcr.io/open-webui/open-terminal@sha256:<verified-digest>'

# Docker (recommended)
docker pull "$OPEN_TERMINAL_IMAGE"
docker run -d --name open-terminal --restart unless-stopped \
  -p 127.0.0.1:8000:8000 \
  -v open-terminal:/home/user \
  -e OPEN_TERMINAL_API_KEY="$OPEN_TERMINAL_API_KEY" \
  "$OPEN_TERMINAL_IMAGE"

# Bare metal through uvx (without installation)
uvx open-terminal run --host 127.0.0.1 --port 8000 --api-key "$OPEN_TERMINAL_API_KEY"

# Bare metal through pip
pip install open-terminal
open-terminal run --host 127.0.0.1 --port 8000 --api-key "$OPEN_TERMINAL_API_KEY"
```

Automatic API key generation in the logs is suitable only for a one-off local test. For a persistent environment, set the key explicitly through an environment variable or a `_FILE` secret.

## Authentication

All endpoints except `/health` require a Bearer token:

```
Authorization: Bearer $OPEN_TERMINAL_API_KEY
```

WebSocket terminals use **first-message authentication**—the first message after connecting must be JSON:
```json
{"type": "auth", "token": "$OPEN_TERMINAL_API_KEY"}
```

## Security Guardrails

- Deploy Open Terminal only behind a VPN, reverse proxy, or allowlist ACL. Do not expose it directly to the internet without a separate isolation boundary.
- For production, prefer Docker, a VM, or a dedicated unprivileged user. Bare metal is appropriate only in a trusted environment.
- Use `/files/upload` with the `url` field, notebook `source`, and `/proxy/{port}/{path}` only with trusted sources. Do not download URLs or execute retrieved content without manual review.
- If you do not need PTYs, notebooks, or the proxy, disable them through configuration or perimeter access controls.
- Do not place real keys in commands, logs, issue trackers, JSON examples, or agent messages.

## Main API Groups

| Group | Purpose |
|-------|---------|
| `POST /execute` | Launch commands as background processes |
| `GET /execute/{id}/status` | Poll output and status |
| `POST /execute/{id}/input` | Send stdin to a process |
| `DELETE /execute/{id}` | Terminate (kill) a process |
| `GET /files/list` | List a directory |
| `GET /files/read` | Read a file (text/PDF/binary) |
| `POST /files/write` | Write a file |
| `POST /files/replace` | Find and replace in a file |
| `GET /files/grep` | Search file contents |
| `GET /files/glob` | Find files by name (glob patterns) |
| `GET /files/display` | Open a file in the user's UI |
| `POST /files/upload` | Upload a file (multipart or URL) |
| `POST /api/terminals` | Create an interactive PTY session |
| `WS /api/terminals/{id}` | Terminal WebSocket |
| `GET /ports` | Detect listening TCP ports |
| `/proxy/{port}/{path}` | Reverse proxy to localhost:{port} |
| `GET /health` | Health check (no authentication) |

For details about each endpoint, see `references/api.md`.

## Command Execution Model

Commands are launched through `POST /execute` as **background processes**:

1. The command is spawned in a PTY (Unix), WinPTY (Windows), or pipes (fallback)
2. It receives a UUID; output is written to a JSONL log in `LOG_DIR`
3. Status transitions from `running` to `done` (or `killed`)
4. Output is read by polling `GET /execute/{id}/status` with `offset` and `tail`
5. Completed processes are automatically removed five minutes after they finish

The `wait` parameter (0–300 seconds) waits for a command to finish inline, which is convenient for short commands.

## Interactive Terminals

PTY sessions are managed through REST and WebSocket:

1. `POST /api/terminals`—create a session (limit: `MAX_TERMINAL_SESSIONS`, default 16)
2. `WS /api/terminals/{id}`—connect (first-message authentication, then binary input/output frames)
3. Resize: text JSON frame `{"type": "resize", "cols": 120, "rows": 40}`
4. Dead sessions are automatically cleaned up when sessions are listed

## Configuration

Precedence, from highest to lowest:
1. CLI flags (`--host`, `--port`, `--api-key`)
2. Environment variables (`OPEN_TERMINAL_*`)
3. User configuration: `~/.config/open-terminal/config.toml`
4. System configuration: `/etc/open-terminal/config.toml`
5. Built-in defaults

For a detailed table of all environment variables and parameters, see `references/configuration.md`.

### Docker-Specific Variables

| Variable | Description |
|----------|-------------|
| `OPEN_TERMINAL_PACKAGES` | Space-separated list of apt packages to install at startup |
| `OPEN_TERMINAL_PIP_PACKAGES` | Space-separated list of pip packages to install at startup |

These are processed by `entrypoint.sh` each time the container starts.

### Docker Secrets

The `_FILE` convention (as used by PostgreSQL) is supported:
- `OPEN_TERMINAL_API_KEY_FILE=/run/secrets/api_key`—the value is read from the file
- `VAR` and `VAR_FILE` cannot be set at the same time

### Docker Socket

To access the Docker CLI from inside the container (building images and launching containers):
```bash
docker run -d -p 8000:8000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v open-terminal:/home/user \
  "$OPEN_TERMINAL_IMAGE"
```
`entrypoint.sh` automatically adds the user to the socket's group.

## Open WebUI Integration

Open Terminal integrates with Open WebUI in two ways:

### Direct Connection (from the Browser)
Users connect their own instance through **User Settings → Integrations → Open Terminal**. Requests go directly from the browser.

### System-Level (through the Backend)
An administrator configures it through **Admin Settings → Integrations → Open Terminal**. Requests are proxied through the Open WebUI backend. Multiple terminals can be configured with per-user and per-group access control.

In both cases, the connection is added under **Open Terminal** (not as a tool server), providing a file sidebar for navigating, uploading, and editing files.

## Codebase Architecture

For information about working with the open-terminal source code, see `references/architecture.md`.

## MCP Server

Open Terminal can run as an MCP (Model Context Protocol) server:

```bash
# Requires installation: pip install open-terminal[mcp]
open-terminal mcp --transport stdio
open-terminal mcp --transport streamable-http --host 0.0.0.0 --port 8000
```

All FastAPI endpoints are automatically exposed as MCP tools through `FastMCP.from_fastapi()`.

## Jupyter Notebooks

Hidden endpoints (not included in the OpenAPI schema) support per-cell notebook execution:

1. `POST /notebooks`—create a session (associate it with an `.ipynb` file and start a kernel)
2. `POST /notebooks/{session_id}/execute`—execute a cell (by index, with an optional source override)
3. `GET /notebooks/{session_id}`—get session status
4. `DELETE /notebooks/{session_id}`—stop the kernel

Sessions are automatically removed after 30 minutes of inactivity. The notebook is saved to disk after each execution.

<!-- A-EVOLVE-ROUTING-SIGNALS:START -->
## Routing signals: open terminal rest api execute files sessions sandbox command notebook self-hosted terminal api agents open_terminal_api_key
<!-- A-EVOLVE-ROUTING-SIGNALS:END -->
