# Open Terminal Codebase Architecture

## Security Posture

Open Terminal provides a legitimate but high-risk set of capabilities: remote command execution, PTY over WebSocket, file writing, notebooks, and a local reverse proxy. It should be treated as a sensitive admin-plane service.

- Deploy it in an isolated container or on a dedicated host.
- Keep access behind strict authentication and a network perimeter.
- Pin the Docker image by digest or release tag rather than using the floating `latest` tag.
- Review `verify_api_key`, the entrypoint, and network access rules before exposing it publicly.

## Project Structure

```
open-terminal/
├── open_terminal/               # Python package
│   ├── __init__.py              # Docstring
│   ├── __main__.py              # Entry point: calls cli.main()
│   ├── cli.py                   # Click CLI: run and mcp commands
│   ├── config.py                # Loads and merges TOML configuration files
│   ├── env.py                   # All environment variables → module-level constants
│   ├── main.py                  # FastAPI app, ALL endpoints (~1,500 lines)
│   ├── runner.py                # Process execution abstraction (PTY/WinPTY/Pipes)
│   ├── mcp_server.py            # MCP: FastMCP.from_fastapi(app)
│   ├── notebooks.py             # Jupyter notebook sessions (APIRouter)
│   └── utils/
│       ├── __init__.py
│       └── port.py              # Port detection and process-tree utilities
├── Dockerfile                   # Python 3.12 + Node.js 22 + tools
├── entrypoint.sh                # Docker entrypoint: secrets, packages, permissions
├── dev.sh                       # Dev: uv run uvicorn ... --reload
├── pyproject.toml               # Metadata and dependencies (hatchling)
├── uv.lock                      # Dependency lockfile
├── CHANGELOG.md                 # Version changelog
└── .github/workflows/
    ├── docker.yml               # Build and push multi-arch Docker image
    └── release.yml              # Automatically create a GitHub Release
```

## Key Modules

### main.py — Application Core

This monolithic file (~1,500 lines) contains:

- **FastAPI app** with CORS middleware
- **Pydantic models**: ExecRequest, WriteRequest, ReplaceRequest, ReplacementChunk, MoveRequest, InputRequest, MkdirRequest
- **Authentication**: `verify_api_key()` — Bearer scheme via HTTPBearer
- **Middleware**: `normalize_null_query_params` — removes query parameters whose value is `"null"`
- **File endpoints** (`/files/*`): list, read, write, replace, grep, glob, upload, mkdir, delete, move, display, view, cwd
- **Execution endpoints** (`/execute`): run, status, input, kill, list
- **Port detection & reverse proxy** (`/ports`, `/proxy/{port}/{path}`)
- **Interactive terminals** (`/api/terminals/*`): PTY session creation, WebSocket, resize

Terminals are registered only when `ENABLE_TERMINAL=true`; the entire block is inside `if ENABLE_TERMINAL:`.

The notebook router is registered similarly: `if ENABLE_NOTEBOOKS:` → `app.include_router(create_notebooks_router(verify_api_key))`.

### runner.py — Process Execution Abstraction

Three `ProcessRunner` (ABC) implementations:

| Class | Platform | Mechanism |
|-------|-----------|----------|
| `PtyRunner` | Unix | `pty.openpty()` + `subprocess.Popen` via the slave fd |
| `WinPtyRunner` | Windows | `pywinpty.PtyProcess.spawn()` via ConPTY |
| `PipeRunner` | Any (fallback) | `asyncio.create_subprocess_shell` with PIPE |

The `create_runner()` factory selects the appropriate implementation:
1. If `pty` is available (Unix) → `PtyRunner`
2. If `winpty` is available (Windows) → `WinPtyRunner`
3. Otherwise → `PipeRunner`

Shared interface: `read_output()`, `write_input()`, `kill()`, `wait()`, `close()`, `pid`.

### config.py — Configuration

- `load_config(explicit_path)` — loads and merges system and user TOML configuration files
- `init()` — called once by the CLI at startup and caches the result
- `get(key, default)` — looks up a value in the cached configuration

### env.py — Environment Variables

Module-level constants evaluated at import time:
`API_KEY`, `CORS_ALLOWED_ORIGINS`, `LOG_DIR`, `BINARY_FILE_MIME_PREFIXES`, `MAX_TERMINAL_SESSIONS`, `ENABLE_TERMINAL`, `TERMINAL_TERM`, `EXECUTE_TIMEOUT`, `EXECUTE_DESCRIPTION`, `ENABLE_NOTEBOOKS`.

Each constant follows this resolution chain: environment variable → config.get() → default. Docker secrets are supported through `_resolve_file_env()`.

### notebooks.py — Jupyter Sessions

- `create_notebooks_router(verify_api_key)` — APIRouter factory with the `/notebooks` prefix
- Sessions are stored in `_sessions: dict[str, _Session]`
- Each session consists of a NotebookClient, kernel, and notebook object
- Idle cleanup: a background task runs every minute and removes sessions older than 30 minutes
- The notebook is saved to disk whenever a cell is executed

### utils/port.py — Port Utilities

- `detect_listening_ports()` — cross-platform detector for TCP LISTEN ports
  - Linux: `/proc/net/tcp` + `/proc/net/tcp6` + inode→PID via `/proc/*/fd/`
  - macOS: `lsof -iTCP -sTCP:LISTEN`
  - Windows: `netstat -ano -p tcp`
- `get_descendant_pids(root_pid)` — descendant process tree
  - Linux: `/proc/*/stat`
  - Fallback: `ps -eo pid,ppid`

## Development

```bash
# Start the development server with auto-reload
./dev.sh  # = uv run uvicorn open_terminal.main:app --reload

# Install dependencies
uv sync

# Python 3.11 (.python-version)
# Build system: hatchling
```

The repository has no tests. API documentation is generated automatically from the FastAPI descriptions (Swagger UI at `/docs`).

## CI/CD

- **docker.yml**: push to main → build linux/amd64 + linux/arm64 → push `ghcr.io/open-webui/open-terminal` with the tags `latest`, `<version>`, `<major.minor>`, `sha-*`
- **release.yml**: change to `pyproject.toml` on main → extract changelog → create git tag + GitHub Release

The version is read from `pyproject.toml` → `project.version`.

## Dependencies

**Runtime** (pyproject.toml):
- fastapi, uvicorn[standard] — HTTP server
- click — CLI
- httpx — HTTP client (for URL uploads and the port proxy)
- python-multipart — file uploads
- aiofiles — asynchronous file I/O
- pypdf — PDF text extraction
- nbclient, ipykernel — Jupyter notebook execution
- pywinpty (Windows only) — ConPTY

**Optional**: `fastmcp>=2.0.0` (extra `mcp`)

**The Docker image also includes**:
- Node.js 22 LTS, Docker CLI + Compose + Buildx
- numpy, pandas, scipy, scikit-learn, matplotlib, seaborn, plotly
- jupyter, ipython, requests, beautifulsoup4, sqlalchemy
- openpyxl, weasyprint, python-docx, python-pptx, pypdf, csvkit
- ffmpeg, pandoc, imagemagick, texlive-latex-base
- vim, nano, git, curl, wget, jq, sqlite3, and others.
