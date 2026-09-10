# Project `paseo.json` reference

The per-project filename is lowercase `paseo.json` at the project root. It configures worktree lifecycle, workspace scripts and services, startup terminals, project service-port allocation, and metadata wording.

The canonical schema is `PaseoConfigSchema` in `packages/protocol/src/paseo-config-schema.ts`. Runtime consumers are in `packages/server/src/utils/worktree.ts` and `packages/server/src/server/worktree-bootstrap.ts`.

## Complete shape

```json
{
  "worktree": {
    "setup": ["npm ci", "npm run build"],
    "teardown": "npm run db:drop || true",
    "terminals": [
      { "name": "logs", "command": "tail -f dev.log" },
      { "command": "bash" }
    ],
    "servicePorts": {
      "range": "3000-4000",
      "portScript": "/usr/local/bin/allocate-paseo-port"
    }
  },
  "scripts": {
    "test": {
      "command": "npm test"
    },
    "web": {
      "type": "service",
      "command": "npm run dev -- --port $PASEO_PORT",
      "port": 3000
    }
  },
  "metadataGeneration": {
    "title": {
      "instructions": "Use a short noun phrase."
    },
    "branchName": {
      "instructions": "Use <type>/<scope>-<description>."
    },
    "commitMessage": {
      "instructions": "Follow Conventional Commits."
    },
    "pullRequest": {
      "instructions": "Include a Testing section."
    }
  }
}
```

Every root field is optional.

## Discovery and lifecycle

- Paseo resolves the file at the registered project root. A registered project may be a nested directory inside a larger Git repository.
- Worktree creation reads the committed `paseo.json` from the selected base ref. Uncommitted edits in another checkout do not define a new worktree's policy.
- Paseo seeds the selected file into a new worktree before running setup.
- Existing workspace scripts are read from that workspace's current `paseo.json`.
- App writes use the file's modification time and size as a revision. A concurrent edit returns `stale_project_config` instead of overwriting the newer file.
- Invalid JSON makes runtime parsing fail. Some display/status paths degrade to no scripts and log the parse error; lifecycle operations fail explicitly.
- The app's raw edit API rejects schema-invalid `servicePorts`. Runtime loading is more tolerant: it normalizes an invalid worktree block to empty lifecycle settings instead of executing it.

Commit `paseo.json` when the project should carry the behavior across machines and into new worktrees. Treat it as executable code during review.

## `worktree.setup`

Type: `string | string[]`.

Runs after Paseo creates and seeds a new worktree. Use it to install dependencies, build generated artifacts, copy ignored local files, or initialize per-worktree resources.

```json
{
  "worktree": {
    "setup": [
      "npm ci",
      "node ./scripts/seed-local-config.mjs",
      "npm run build"
    ]
  }
}
```

A string is one shell invocation. It may contain newlines, pipelines, and shell control operators. An array runs each non-empty command sequentially and stops at the first failure.

Setup runs with the worktree as `cwd`. On failure, worktree creation fails and can clean up the new worktree.

## `worktree.teardown`

Type: `string | string[]`.

Runs during archive, before Paseo removes the worktree directory.

```json
{
  "worktree": {
    "teardown": "npm run db:drop || true"
  }
}
```

It uses the same command normalization and shell behavior as setup. A failure aborts teardown/removal, so make cleanup idempotent and handle acceptable missing-resource failures explicitly.

## Lifecycle shell behavior

Setup and teardown run through a stable string-command shell:

- macOS/Linux: `bash` from `PATH`, non-login and non-interactive
- Windows: PowerShell with `-NoProfile`

They inherit the daemon environment plus Paseo lifecycle variables. Bash's `BASH_ENV` hook is unset. User login and interactive startup files are not loaded.

A lifecycle command intended for every platform cannot depend on POSIX-only syntax such as `VAR=1 command`, `$VAR`, `cp`, `rm`, or a shell-script entrypoint. Put cross-platform logic in a Node script and run `node ./scripts/name.mjs`.

## `worktree.terminals`

Runtime shape: array of objects.

| Entry key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `command` | non-empty `string` | Yes | Command sent to the new terminal. |
| `name` | non-empty `string` | No | Terminal name. |

```json
{
  "worktree": {
    "terminals": [
      { "name": "logs", "command": "tail -f dev.log" },
      { "name": "repl", "command": "npm run repl" }
    ]
  }
}
```

These terminals open automatically for newly created worktree workspaces. Invalid entries, missing commands, non-string commands, and blank commands are ignored. Unknown entry fields are ignored.

## `worktree.servicePorts`

Project service-port allocation replaces the global `worktrees.servicePorts` block for this project.

| Key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `range` | TCP range string | One of range/script | Inclusive range such as `3000-4000`. |
| `portScript` | non-empty `string` | One of range/script | Executable external allocator. Wins if both are present. |

The object is strict and must contain at least one of the two keys.

### Range validation

- Format is exactly one to five digits, a hyphen, then one to five digits.
- Start and end must be within 1–65535.
- Start must be less than or equal to end.
- Paseo chooses an available, non-reserved port in the range.
- Allocation fails when no port is available.

```json
{
  "worktree": {
    "servicePorts": { "range": "3000-4000" }
  }
}
```

### External allocator

```json
{
  "worktree": {
    "servicePorts": {
      "portScript": "/usr/local/bin/portmake"
    }
  }
}
```

Paseo executes the configured path directly, without a shell, from the workspace directory. Use a binary or an executable script with a valid shebang. Inline shell commands and pipelines do not work.

Arguments, in order:

1. Service script name
2. Workspace ID
3. Branch name, or an empty string
4. Worktree/workspace path

The same values are available as:

- `PASEO_SCRIPTNAME`
- `PASEO_WORKSPACE_ID`
- `PASEO_BRANCH_NAME`
- `PASEO_WORKTREE_PATH`

The allocator has 10 seconds, may write at most 1 KiB, and must print exactly one integer from 1–65535 to stdout. The result cannot collide with a port reserved for another declared service in the workspace. Paseo otherwise trusts the allocator and does not require the returned port to be free, which permits attaching to an existing service.

Precedence:

1. `scripts.<name>.port`
2. Project `worktree.servicePorts`
3. Global `worktrees.servicePorts`
4. OS ephemeral allocation

## `scripts`

`scripts` is a record keyed by the name displayed and used by the app, CLI, and MCP tools.

```json
{
  "scripts": {
    "test": { "command": "npm test" },
    "web": {
      "type": "service",
      "command": "npm run dev -- --port $PASEO_PORT"
    }
  }
}
```

Runtime fields:

| Entry key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `command` | non-empty `string` | Yes | Command sent to a persistent terminal shell. |
| `type` | literal `"service"` for services | No | Exact `service` enables supervision, port allocation, and proxying. Omitted or any other value behaves as a plain script. |
| `port` | finite `number` | No | Explicit service port. Only used when `type` is `service`. |

Configure explicit ports as TCP integers from 1–65535. Duplicate explicit ports in one workspace are rejected. The raw schema only checks that `port` becomes a finite number at runtime; malformed finite values can survive parsing and fail later when the route or service starts.

The raw forward-compatible schema accepts unknown field types and unknown keys, but runtime only recognizes the values above. A script with a non-string/blank `command` does not exist at runtime. Do not rely on permissive parsing as validation.

### Plain scripts

Plain scripts run on demand in a persistent workspace terminal and transition to stopped when the shell reports command completion.

```json
{
  "scripts": {
    "test": { "command": "npm test" },
    "lint": { "command": "npm run lint" },
    "codegen": { "command": "npm run codegen" }
  }
}
```

### Services

Services are long-running commands supervised through their terminal lifecycle. Paseo allocates or reserves a port, registers a reverse-proxy route, and removes the route when the process stops.

```json
{
  "scripts": {
    "web": {
      "type": "service",
      "command": "npm run dev -- --host $HOST --port $PASEO_PORT"
    },
    "api": {
      "type": "service",
      "command": "PORT=$PASEO_PORT npm run api",
      "port": 3100
    }
  }
}
```

Services do not auto-start merely because they are declared. Start them from the app, CLI, or MCP:

```bash
paseo script ls --workspace <workspace-id>
paseo script start web --workspace <workspace-id>
paseo script stop web --workspace <workspace-id>
```

Service proxy hostname shape:

```text
<script>--<branch>--<project>.localhost
<script>--<project>.localhost        # main/master
```

The proxy supports HTTP and WebSocket upgrades.

## Command environment

Setup, teardown, terminals, and workspace commands operate in the workspace context. Relevant lifecycle variables include:

| Variable | Meaning |
| --- | --- |
| `PASEO_SOURCE_CHECKOUT_PATH` | Original source checkout root. |
| `PASEO_ROOT_PATH` | Backward-compatible alias of source checkout path. |
| `PASEO_WORKTREE_PATH` | Current worktree/workspace path. |
| `PASEO_BRANCH_NAME` | Current branch name. |
| `PASEO_WORKTREE_PORT` | Legacy per-worktree port. Prefer service-specific variables. |

Every service receives:

| Variable | Meaning |
| --- | --- |
| `PASEO_PORT` | This service's assigned port. |
| `PASEO_URL` | This service's proxied URL. |
| `PASEO_SERVICE_<NAME>_PORT` | Assigned port for each declared service, including itself. |
| `PASEO_SERVICE_<NAME>_URL` | Stable proxied URL for each declared service. |
| `HOST` | Bind host based on daemon exposure (`127.0.0.1` or `0.0.0.0`). |

`PORT` is not injected. Set it in the command if the framework requires it.

For service environment names, Paseo uppercases the script name, changes each run of non-`A-Z0-9` characters to `_`, and trims leading/trailing underscores. `app-server` and `app.server` both become `APP_SERVER`; such collisions fail when a service starts.

Use peer `_URL` values for service discovery. A peer's raw `_PORT` can change after restart.

## `metadataGeneration`

Per-project instructions replace Paseo's default style text for each metadata kind. Functional output requirements still apply.

| Key | Entry shape | Purpose |
| --- | --- | --- |
| `metadataGeneration.title` | `{ "instructions"?: string }` | Workspace title wording. |
| `metadataGeneration.branchName` | `{ "instructions"?: string }` | New worktree branch naming. |
| `metadataGeneration.commitMessage` | `{ "instructions"?: string }` | Generated commit message wording. |
| `metadataGeneration.pullRequest` | `{ "instructions"?: string }` | Generated PR title/body wording. |

```json
{
  "metadataGeneration": {
    "title": {
      "instructions": "Keep titles to four words."
    },
    "branchName": {
      "instructions": "Use feat/, fix/, docs/, or chore/ prefixes."
    },
    "commitMessage": {
      "instructions": "Follow Conventional Commits with no trailing period."
    },
    "pullRequest": {
      "instructions": "Include Summary and Testing sections."
    }
  }
}
```

Each `instructions` value is a string. Empty or whitespace-only text is treated as no useful override by consumers and should be removed.

Project metadata wording is separate from global `agents.metadataGeneration.providers`:

- Global config chooses the ordered provider/model candidates.
- Project config changes the task-specific writing instructions.

The removed legacy `metadataGeneration.agentTitle` key may survive a passthrough round trip for compatibility but has no current behavior. Use `title`.

## Forward compatibility and unknown fields

The project root, `worktree`, script entries, `metadataGeneration`, and metadata entries preserve unknown keys. This allows newer files to pass through older edit clients.

That permissiveness does not make unknown fields functional:

- Runtime ignores unknown root and worktree fields.
- Runtime ignores malformed terminal entries.
- Runtime ignores script entries without a non-empty string `command`.
- Only exact `type: "service"` creates a service.
- Invalid metadata entries normalize to empty entries.
- `servicePorts` is strict in the raw edit schema. Invalid values make app reads/writes return `invalid_project_config`; tolerant runtime loading drops the invalid worktree settings.

When editing through Paseo's Project settings UI, unknown sibling fields are preserved where possible.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| New worktree used old setup | Paseo read the selected base ref's committed file | Commit the intended `paseo.json` on the base ref or select the correct base |
| File seems ignored | Wrong case or wrong registered project root | Use lowercase `paseo.json` at the exact project root |
| Script is missing | `command` is absent, blank, or not a string | Use one non-empty command string |
| Script exits but stays listed | Script state includes stopped entries and terminal history | Inspect lifecycle/exit code; restart it through script controls |
| No proxy URL | `type` is not exactly `service` | Set `"type": "service"` |
| Framework bound the wrong port | It expects `PORT`, which Paseo does not inject | Set `PORT=$PASEO_PORT` or pass `$PASEO_PORT` using the framework's flag |
| Port allocation fails | Invalid/exhausted range, duplicate explicit port, allocator timeout, or invalid output | Fix the allocation block or executable; remember explicit service ports win |
| Peer URL env is absent | Two script names normalize to the same environment key or the peer is not a valid service | Rename colliding scripts and ensure each service has a valid command |
| Setup works on macOS but not Windows | POSIX-specific command in a cross-platform project | Move logic into a Node script |
| App refuses to save | File changed after the app read it | Reload Project settings and reapply the edit; do not overwrite the newer disk version |
| Metadata style did not change | Wrong metadata key, empty instructions, or generation used another project's root | Use `title`, `branchName`, `commitMessage`, or `pullRequest` at the registered root |
