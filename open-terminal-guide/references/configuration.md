# Open Terminal Configuration

## Configuration Precedence

1. **CLI flags** — `--host`, `--port`, `--api-key`, `--cors-allowed-origins`, `--config`, `--cwd`
2. **Environment variables** — `OPEN_TERMINAL_*`
3. **User config** — `$XDG_CONFIG_HOME/open-terminal/config.toml` (defaults to `~/.config/open-terminal/config.toml`)
4. **System config** — `/etc/open-terminal/config.toml`
5. **Built-in defaults**

## Environment Variables

| Variable | TOML key | Default | Description |
|-----------|-----------|-----------|----------|
| `OPEN_TERMINAL_API_KEY` | `api_key` | *(auto-generated)* | Bearer token for authentication |
| `OPEN_TERMINAL_CORS_ALLOWED_ORIGINS` | `cors_allowed_origins` | `*` | Allowed CORS origins (comma-separated) |
| `OPEN_TERMINAL_LOG_DIR` | `log_dir` | `~/.local/state/open-terminal/logs` | Directory for process JSONL logs |
| `OPEN_TERMINAL_BINARY_MIME_PREFIXES` | `binary_mime_prefixes` | `image` | MIME prefixes for binary files in /files/read (comma-separated) |
| `OPEN_TERMINAL_MAX_SESSIONS` | `max_terminal_sessions` | `16` | Maximum number of concurrent PTY sessions |
| `OPEN_TERMINAL_ENABLE_TERMINAL` | `enable_terminal` | `true` | Enable interactive terminals |
| `OPEN_TERMINAL_TERM` | `term` | `xterm-256color` | $TERM value for PTY sessions |
| `OPEN_TERMINAL_EXECUTE_TIMEOUT` | `execute_timeout` | *(not set)* | Default timeout for `wait` in /execute (seconds) |
| `OPEN_TERMINAL_EXECUTE_DESCRIPTION` | `execute_description` | *(empty)* | Additional text for the /execute description in OpenAPI |
| `OPEN_TERMINAL_ENABLE_NOTEBOOKS` | `enable_notebooks` | `true` | Enable Jupyter notebook sessions |

### Docker-Only Variables

| Variable | Description |
|-----------|----------|
| `OPEN_TERMINAL_PACKAGES` | Space-separated list of apt packages to install when the container starts |
| `OPEN_TERMINAL_PIP_PACKAGES` | Space-separated list of pip packages to install at startup |

These variables are processed in `entrypoint.sh`, not in the Python code.

## Docker Secrets (`_FILE` Convention)

Open Terminal supports the `_FILE` suffix for securely passing secrets:

```yaml
# docker-compose.yml
services:
  terminal:
    image: ghcr.io/open-webui/open-terminal@sha256:<verified-digest>
    secrets:
      - api_key
    environment:
      OPEN_TERMINAL_API_KEY_FILE: /run/secrets/api_key

secrets:
  api_key:
    file: ./api_key.txt
```

This is implemented in two places:
- `entrypoint.sh` — the `file_env` function for the shell layer
- `open_terminal/env.py` — the `_resolve_file_env()` function for the Python layer

Setting both `VAR` and `VAR_FILE` is not allowed and results in an error.

## Example TOML Configuration

```toml
host = "127.0.0.1"
port = 8000
api_key = "<read-from-secret-store>"
cors_allowed_origins = "*"
log_dir = "/var/log/open-terminal"
binary_mime_prefixes = "image,audio"
execute_timeout = 5
max_terminal_sessions = 32
enable_terminal = true
enable_notebooks = true
term = "xterm-256color"
```

## CLI Commands

### `open-terminal run`

Starts the REST API server (uvicorn).

```bash
open-terminal run [OPTIONS]
  --host TEXT          Bind host (default: 0.0.0.0)
  --port INT           Bind port (default: 8000)
  --config PATH        Path to the TOML configuration file
  --cwd PATH           Working directory
  --api-key TEXT       Bearer API key
  --cors-allowed-origins TEXT  Comma-separated CORS origins
```

### `open-terminal mcp`

Starts the MCP server (requires `pip install open-terminal[mcp]`).

```bash
open-terminal mcp [OPTIONS]
  --transport [stdio|streamable-http]  Transport (default: stdio)
  --host TEXT          Bind host (streamable-http only)
  --port INT           Bind port (streamable-http only)
  --config PATH        Path to the TOML configuration file
  --cwd PATH           Working directory
```
