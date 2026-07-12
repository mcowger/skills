# API Reference — Open Terminal

The complete interactive Swagger documentation is available at `http://<host>:<port>/docs`.

All endpoints except `/health` require `Authorization: Bearer $OPEN_TERMINAL_API_KEY`.

## Security Guardrails

- Treat `command`, `url`, notebook `source`, uploaded file contents, and proxied HTTP traffic as untrusted input.
- Do not download an external URL through `/files/upload` and pass it to `/execute`, notebook execution, or a shell without separately validating its contents and domain.
- `/proxy/{port}/{path}` exposes local services accessible to the process. Do not enable this route for untrusted users or expose internal admin UIs through it.
- Do not include real Bearer tokens in examples, curl commands, logs, or tickets.

## Table of Contents

- [Execute (Commands)](#execute-commands)
- [Files (File Operations)](#files-file-operations)
- [Terminals (Interactive PTYs)](#terminals-interactive-ptys)
- [Notebooks (Jupyter)](#notebooks-jupyter)
- [Ports & Proxy](#ports--proxy)
- [Health & Config](#health--config)

---

## Execute (Commands)

### POST /execute — Run a Command

Runs a shell command as a background process.

**Body (JSON):**
```json
{
  "command": "ls -la && whoami",
  "cwd": "/home/user/project",
  "env": {"MY_VAR": "value"}
}
```

**Query parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `wait` | float (0–300) | Number of seconds to wait for completion. If the command finishes in time, return its output inline. `null` returns immediately |
| `tail` | int (≥1) | Return only the last N output records |

If `wait` is not specified but `EXECUTE_TIMEOUT` is configured, that value is used.

**Response:**
```json
{
  "id": "a1b2c3d4e5f6",
  "command": "ls -la",
  "status": "running",
  "exit_code": null,
  "output": [{"type": "output", "data": "..."}],
  "truncated": false,
  "next_offset": 5,
  "log_path": "/path/to/log.jsonl"
}
```

### GET /execute/{process_id}/status — Poll Status

**Query parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `wait` | float (0–300) | Wait for completion before responding |
| `offset` | int (≥0) | Skip N records (use `next_offset` from the previous response) |
| `tail` | int (≥1) | Return only the last N records |

Polling pattern: call with `offset=0`, retrieve `next_offset`, then make the next call with `offset=next_offset`.

### POST /execute/{process_id}/input — Send Input

```json
{"input": "yes\n"}
```

Escape sequences (`\n`, `\x03` for Ctrl-C) are automatically converted from literal strings.

### DELETE /execute/{process_id} — Terminate a Process

| Parameter | Type | Description |
|-----------|------|-------------|
| `force` | bool | `true` = SIGKILL, `false` (default) = SIGTERM |

### GET /execute — List Processes

Returns all tracked processes (running, done, killed). Completed processes are automatically removed after 5 minutes.

---

## Files (File Operations)

### GET /files/list — List a Directory

| Parameter | Type | Description |
|-----------|------|-------------|
| `directory` | string | Path (default: `.`) |

Response: `{dir, entries: [{name, type, size, modified}]}`.

### GET /files/read — Read a File

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | string | File path (required) |
| `start_line` | int (≥1) | Starting line (1-indexed) |
| `end_line` | int (≥1) | Ending line (1-indexed) |

Behavior depends on the file type:
- **Text** → `{path, total_lines, content}`
- **PDF** → extracts text with pypdf and returns it as text
- **Images** (image/*) → raw binary with Content-Type
- **Other binary files** → 415 Unsupported

The MIME types returned as binary are configured through `OPEN_TERMINAL_BINARY_MIME_PREFIXES` (default: `image`).

### GET /files/display — Display a File to the User

Does not return content! Signals the client (Open WebUI) to open the file in its UI.

### GET /files/view — View a File (Raw)

Returns any file as-is with the correct Content-Type. Used by clients for previews (not for the LLM).

### POST /files/write — Write a File

```json
{"path": "/home/user/hello.txt", "content": "Hello world"}
```
Parent directories are created automatically. An existing file is overwritten.

### POST /files/replace — Find and Replace

```json
{
  "path": "/home/user/code.py",
  "replacements": [
    {
      "target": "old_function_name",
      "replacement": "new_function_name",
      "start_line": 10,
      "end_line": 50,
      "allow_multiple": false
    }
  ]
}
```

- `start_line` / `end_line` — narrow the search range (1-indexed, optional)
- `allow_multiple: false` (default) — raises an error if more than one occurrence is found
- Replacements are applied sequentially

### GET /files/grep — Search File Contents

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | string | Text or regex |
| `path` | string | Directory/file (default: `.`) |
| `regex` | bool | Treat query as a regex |
| `case_insensitive` | bool | Perform a case-insensitive search |
| `include` | list[string] | Glob filters (`*.py`) |
| `match_per_line` | bool | `true` = lines with line numbers, `false` = file names only |
| `max_results` | int (1–500) | Result limit (default: 50) |

### GET /files/glob — Search by Name

| Parameter | Type | Description |
|-----------|------|-------------|
| `pattern` | string | Glob pattern (`*.py`) |
| `path` | string | Directory (default: `.`) |
| `exclude` | list[string] | Exclusion patterns |
| `type` | string | `file`, `directory`, `any` |
| `max_results` | int (1–500) | Limit (default: 50) |

### POST /files/upload — Upload a File

| Parameter | Type | Description |
|-----------|------|-------------|
| `directory` | string | Target directory (required) |
| `url` | string | URL to download (optional) |
| `file` | UploadFile | Multipart upload (if no URL is provided) |

If you use `url`, restrict it to an allowlist of domains and validate the content type and size first. Do not consider an uploaded file safe merely because the server downloaded it.

### POST /files/mkdir — Create a Directory
### DELETE /files/delete — Delete a File or Directory Recursively
### POST /files/move — Move/Rename

---

## Terminals (Interactive PTYs)

These endpoints are hidden from the OpenAPI schema (`include_in_schema=False`).

### POST /api/terminals — Create a Session

Creates a PTY process (Unix: pty + shell, Windows: WinPTY + cmd.exe). Limit: `MAX_TERMINAL_SESSIONS` (default: 16).

**Response:**
```json
{"id": "a1b2c3d4", "created_at": "2025-01-01T00:00:00Z", "pid": 12345}
```

### GET /api/terminals — List Active Sessions
### GET /api/terminals/{id} — Get Session Information
### DELETE /api/terminals/{id} — Terminate a Session

### WS /api/terminals/{id} — WebSocket

Protocol:
1. Connect → `accept()`
2. First-message authentication: send `{"type": "auth", "token": "<key>"}`
3. Input: binary frames (keystrokes)
4. Output: binary frames from the PTY
5. Resize: text JSON `{"type": "resize", "cols": N, "rows": M}`

Close codes: `4001` — authentication error, `4004` — session not found.

---

## Notebooks (Jupyter)

All endpoints are hidden from the OpenAPI schema. Idle timeout: 30 minutes.

### POST /notebooks — Create a Session

```json
{"path": "/home/user/analysis.ipynb"}
```

Starts a Jupyter kernel (through nbclient). The kernel is determined from the notebook metadata.

**Response:** `{id, kernel, status: "ready"}`

### POST /notebooks/{session_id}/execute — Execute a Cell

```json
{"cell_index": 0, "source": "print('hello')"}
```

- `cell_index` — 0-based
- `source` — optional; if omitted, the cell's current source is executed. Do not insert code obtained from an untrusted URL or document without manual review.
- The notebook is automatically saved to disk after execution

**Response:** `{status: "ok"|"error", execution_count, outputs: [...]}`

### GET /notebooks/{session_id} — Get Session Status
### DELETE /notebooks/{session_id} — Stop the Kernel

---

## Ports & Proxy

### GET /ports — Discover Ports

Returns TCP ports listened on by child processes of the open-terminal process (started through /execute or a terminal).

Cross-platform: Linux (/proc/net/tcp), macOS (lsof), Windows (netstat).

### /proxy/{port}/{path} — Reverse Proxy

Proxies HTTP requests to `localhost:{port}/{path}`. Supports all methods (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS). Headers are forwarded (excluding hop-by-hop headers).

Use only for trusted local development services. Do not turn `/proxy` into a general-purpose tunnel to internal dashboards or metadata endpoints.

---

## Health & Config

### GET /health — Health Check

```json
{"status": "ok"}
```

Does not require authentication.

### GET /api/config — Feature Flags

```json
{"features": {"terminal": true, "notebooks": true}}
```

Does not require authentication. Used by clients for capability discovery.

### GET /files/cwd — Get the Current Working Directory
### POST /files/cwd — Change the Current Working Directory
