# Global `config.json` reference

This is the complete runtime field catalog for `$PASEO_HOME/config.json` in the reviewed Paseo checkout. `PASEO_HOME` defaults to `~/.paseo`.

The canonical schema is `PersistedConfigSchema` in `packages/server/src/server/persisted-config.ts`. The file is strict JSON. The global root and most nested objects reject unknown fields.

## Minimal initialized file

A missing file is created with private permissions and this content:

```json
{
  "version": 1,
  "daemon": {
    "listen": "127.0.0.1:6767",
    "cors": {
      "allowedOrigins": ["https://app.paseo.sh"]
    },
    "relay": {
      "enabled": false
    }
  },
  "app": {
    "baseUrl": "https://app.paseo.sh"
  }
}
```

All schema fields are optional. Omission applies the runtime fallback; it does not always recreate the value in this initialized file.

## Root fields

| Key | Type | Default / behavior |
| --- | --- | --- |
| `$schema` | `string` | Editor hint. Use `https://paseo.sh/schemas/paseo.config.v1.json`. |
| `version` | literal number `1` | Optional schema marker; write `1` in current configs. |
| `daemon` | object | Daemon networking and behavior. |
| `app` | object | Client application endpoint settings. |
| `worktrees` | object | Global worktree location and service-port policy. |
| `providers` | object | Paseo speech provider credentials and paths. |
| `agents` | object | Coding-agent providers, metadata models, and skill selection. |
| `pluginsEnabled` | `boolean` | `false`; global plugin execution switch. |
| `plugins` | plugin record | Configured plugin installations. |
| `features` | object | Dictation, voice mode, and bundled web UI. |
| `log` | object | Console and file logging. |

## `daemon`

### Networking and host checks

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.listen` | `string` | `127.0.0.1:6767`; accepts `host:port`, a Unix socket path, or `unix:///path`. |
| `daemon.hostnames` | `true | string[]` | Adds accepted Host header names. `true` accepts any hostname. |
| `daemon.allowedHosts` | `true | string[]` | Deprecated alias. Used only when `hostnames` is absent. |
| `daemon.trustedProxies` | `true | string[]` | `['loopback']`; Express proxy names, addresses, or CIDRs. |
| `daemon.cors.allowedOrigins` | `string[]` | Extra allowed browser origins. A newly initialized home includes `https://app.paseo.sh`. |

Hostname behavior:

- `localhost`, `*.localhost`, and IP addresses are always accepted.
- A pattern beginning with `.` accepts that domain and its subdomains.
- Persisted and environment hostname lists append and deduplicate.
- `hostnames: true` short-circuits to allow all Host headers.

CORS origins from the file and `PASEO_CORS_ORIGINS` also append and deduplicate.

### MCP and browser tools

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.mcp.enabled` | `boolean` | `true`; enables the daemon MCP endpoints. |
| `daemon.mcp.injectIntoAgents` | `boolean` | `false`; injects the Paseo tool catalog into new/reloaded agent sessions. |
| `daemon.browserTools.enabled` | `boolean` | `false`; adds browser tools when MCP injection and a browser host are also available. |

The `mcp` and `browserTools` objects are passthrough objects for compatibility, but only the fields above are core runtime settings. Do not invent extra keys.

`agents.providers.<id>.paseoTools` can narrow tool exposure per provider. It cannot re-enable tools when `daemon.mcp.injectIntoAgents` is false.

### Git and lifecycle behavior

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.git.maxProcessesPerSecond` | positive integer | `64`; maximum Git process starts in any one-second interval. |
| `daemon.git.maxProcessConcurrency` | positive integer | `8`; maximum Git child processes running at once. |
| `daemon.autoArchiveAfterMerge` | `boolean` | `false`; archives eligible workspaces after their change request merges. |
| `daemon.enableTerminalAgentHooks` | `boolean` | `false`; installs terminal-agent activity hooks. |
| `daemon.appendSystemPrompt` | `string` | Empty; appended to supported providers' system/developer instructions. |

### Terminal profiles

`daemon.terminalProfiles` is an array of named terminal launches.

| Entry key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `id` | `string` | Yes | Stable profile ID. |
| `name` | `string` | Yes | Display name. |
| `command` | `string` | Yes | Executable or command string used by terminal launch. |
| `args` | `string[]` | No | Arguments. Use `{{{prompt}}}` where typed prompt text belongs. |
| `icon` | `string` | No | Key in Paseo's icon registry. Unknown values use the fallback. |

```json
{
  "daemon": {
    "terminalProfiles": [
      {
        "id": "shell",
        "name": "Shell",
        "command": "bash"
      },
      {
        "id": "codex",
        "name": "Codex",
        "command": "codex",
        "args": ["{{{prompt}}}"],
        "icon": "codex"
      }
    ]
  }
}
```

The array replaces the whole profile list. Omit it to use built-in Claude Code, Codex, OpenCode, and Pi profiles. Set `[]` to remove all terminal profiles. Profile entry objects are passthrough for protocol compatibility, but only the keys above have defined behavior.

### Agent profiles

`daemon.agentProfiles` is an array of named launch bundles.

| Entry key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `id` | `string` | Yes | Stable profile ID. |
| `name` | `string` | Yes | Display name. |
| `provider` | `string` | Yes | Exact built-in or custom provider ID. |
| `icon` | `string` | No | Icon registry key. |
| `color` | `string` | No | Identity color name used by host badges. |
| `model` | `string` | No | Provider model ID. |
| `modeId` | `string` | No | Provider mode ID. |
| `thinkingOptionId` | `string` | No | Provider/model thinking option ID. |
| `featureValues` | `Record<string, unknown>` | No | Provider feature values keyed by feature ID. |
| `notes` | `string` | No | Free text exposed to orchestrators by `list_profiles`. |

```json
{
  "daemon": {
    "agentProfiles": [
      {
        "id": "review",
        "name": "Review",
        "provider": "codex",
        "model": "gpt-5.4",
        "modeId": "read-only",
        "thinkingOptionId": "high",
        "featureValues": { "fast_mode": false },
        "notes": "Use for independent diff and correctness review."
      }
    ]
  }
}
```

The array replaces the whole list. Agent profiles have no built-in defaults. A profile carries no system prompt because per-agent system prompts only apply at creation. Entry objects are passthrough, but only the listed keys have defined behavior.

### Relay

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.relay.enabled` | `boolean` | New homes write `false`. Set explicitly in older homes to avoid legacy fallback behavior. |
| `daemon.relay.endpoint` | `string` | `relay.paseo.sh:443`; daemon's relay connection target. |
| `daemon.relay.publicEndpoint` | `string` | Falls back to `endpoint`; endpoint advertised to clients. |
| `daemon.relay.useTls` | `boolean` | Defaults true for the hosted default endpoint, otherwise false. |
| `daemon.relay.publicUseTls` | `boolean` | Falls back to `useTls`. |

### Service proxy

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.serviceProxy.enabled` | `boolean` | Compatibility shim. `false` suppresses optional listen/public layers only. Localhost proxying remains enabled. |
| `daemon.serviceProxy.listen` | `string` | Optional separate service-only listener, such as `0.0.0.0:8080`. |
| `daemon.serviceProxy.publicBaseUrl` | URI string | Optional public base URL for generated service aliases. |

```json
{
  "daemon": {
    "serviceProxy": {
      "listen": "0.0.0.0:8080",
      "publicBaseUrl": "https://paseoapps.example.com"
    }
  }
}
```

A reverse proxy must preserve the Host header. Public service URLs expose the workspace service itself; daemon password auth does not protect proxied development services.

### Authentication

| Key | Type | Default / behavior |
| --- | --- | --- |
| `daemon.auth.password` | bcrypt hash string | No password by default. Must match a `$2a$`, `$2b$`, or `$2y$` bcrypt hash. |

Use `paseo daemon set-password`. `PASEO_PASSWORD` accepts plaintext at launch and Paseo hashes it in memory; never write plaintext into `daemon.auth.password`.

## `app`

| Key | Type | Default / behavior |
| --- | --- | --- |
| `app.baseUrl` | `string` | `https://app.paseo.sh`; application URL used by daemon flows. |

## `worktrees`

| Key | Type | Default / behavior |
| --- | --- | --- |
| `worktrees.root` | non-empty `string` | `$PASEO_HOME/worktrees`; relative paths resolve against `$PASEO_HOME`, `~` expands. Existing worktrees do not move. |
| `worktrees.servicePorts` | allocation object | OS ephemeral ports when omitted. |
| `worktrees.servicePorts.range` | `"start-end"` | Inclusive TCP range; both ends must be 1–65535 and start must be no greater than end. |
| `worktrees.servicePorts.portScript` | non-empty `string` | Executable allocator. Wins over `range` when both are present. |

An allocation object must contain `range` or `portScript`. Project `worktree.servicePorts` replaces this block for that project. A service's explicit `port` wins over either.

`portScript` runs directly, without a shell, in the workspace directory. It receives service name, workspace ID, branch name, and worktree path as argv and matching `PASEO_*` environment variables. It has 10 seconds, may print at most 1 KiB, and must print exactly one valid, non-reserved TCP port.

## Paseo speech `providers`

This root `providers` object configures Paseo's own speech traffic. It is separate from coding-agent definitions under `agents.providers`.

### `providers.openai`

| Key | Type | Resolution behavior |
| --- | --- | --- |
| `providers.openai.apiKey` | non-empty `string` | Shared fallback credential for STT and TTS. |
| `providers.openai.baseUrl` | trimmed non-empty `string` | Shared fallback endpoint. |
| `providers.openai.stt.apiKey` | trimmed non-empty `string` | STT-specific credential; highest persisted priority. |
| `providers.openai.stt.baseUrl` | trimmed non-empty `string` | STT-specific endpoint. |
| `providers.openai.tts.apiKey` | trimmed non-empty `string` | TTS-specific credential; highest persisted priority. |
| `providers.openai.tts.baseUrl` | trimmed non-empty `string` | TTS-specific endpoint. |

Credential order per endpoint is endpoint-specific persisted key, endpoint-specific environment key, shared persisted key, then `OPENAI_API_KEY`. Base URLs follow the same shape with `OPENAI_*_BASE_URL` and `OPENAI_BASE_URL`.

Prefer environment variables to storing keys in JSON.

### `providers.local`

| Key | Type | Default / behavior |
| --- | --- | --- |
| `providers.local.modelsDir` | non-empty `string` | `$PASEO_HOME/models/local-speech`; local STT/TTS model storage. |

The loader discards removed legacy `providers.local.autoDownload` and `providers.openai.voice` fields. Use `providers.openai.stt` and `.tts`.

## `agents`

### Coding-agent provider overrides

`agents.providers` is a record keyed by provider ID. Built-in IDs are:

- `claude`
- `codex`
- `copilot`
- `opencode`
- `pi`
- `omp`

Custom IDs must match `^[a-z][a-z0-9-]*$`, declare `extends`, and declare `label`. Valid bases are the built-ins plus `acp`. A provider extending `acp` must also declare `command`.

| Provider key | Type | Meaning |
| --- | --- | --- |
| `extends` | `string` | Built-in provider ID or `acp`. Required for custom providers. |
| `label` | `string` | UI label. Required for custom providers. |
| `description` | `string` | UI description. |
| `command` | non-empty `string[]` | Complete launch argv; replaces the built-in command. Required for ACP. |
| `env` | `Record<string, string>` | Literal environment overlay for the provider process. |
| `params` | `Record<string, unknown>` | Provider-specific options; see below. |
| `models` | model array | Static model catalog that replaces runtime-discovered models. |
| `additionalModels` | model array | Adds models or updates discovered models by ID. |
| `disallowedTools` | `string[]` | Provider-native tools to suppress, such as Claude `WebSearch`. |
| `paseoTools` | object | Per-provider Paseo tool policy. |
| `enabled` | `boolean` | Hides/disables the provider when false. Most built-ins default on; OMP defaults off. |
| `order` | `number` | Provider list sort order. |

Provider model entries:

| Model key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `id` | non-empty `string` | Yes | Model identifier sent to the provider. |
| `label` | non-empty `string` | Yes | Display label. |
| `description` | `string` | No | Display description. |
| `isDefault` | `boolean` | No | Preferred model in this catalog. |
| `thinkingOptions` | array | No | Supported reasoning choices. |

Thinking option entries:

| Key | Type | Required |
| --- | --- | --- |
| `id` | `string` | Yes |
| `label` | `string` | Yes |
| `description` | `string` | No |
| `isDefault` | `boolean` | No |

Per-provider Paseo tool policy:

| Key | Type | Default / behavior |
| --- | --- | --- |
| `paseoTools.enabled` | `boolean` | Enabled when omitted. False removes the Paseo catalog for this provider. |
| `paseoTools.disabledTools` | `string[]` | Exact core/browser tool IDs to omit. |

The policy is catalog filtering, not an OS authorization boundary. It does not affect the voice-only `speak` tool.

#### Provider-specific `params`

The schema accepts an arbitrary record so each adapter can validate its own options. Supported documented core options are:

| Provider/base | Param | Type | Default / meaning |
| --- | --- | --- | --- |
| `acp` | `supportsMcpServers` | `boolean` | Defaults to ACP capability behavior. False sends no injected MCP servers. |
| `acp` | `clientCapabilities.fs.readTextFile` | `boolean` | Whether Paseo offers filesystem reads to the ACP agent. |
| `acp` | `clientCapabilities.fs.writeTextFile` | `boolean` | Whether Paseo offers filesystem writes to the ACP agent. |
| `acp` | `clientCapabilities.terminal` | `boolean` | Whether Paseo offers terminal operations. False keeps terminal execution agent-side. |
| `pi` | `sessionDir` | non-empty `string` | Import-only JSONL session directory for Pi-compatible forks. |
| `pi` | `rpcTimeoutMs` | positive integer | `60000`; Pi control-plane RPC deadline. |
| `pi` | `extensionTimeoutMs` | positive integer | `30000`; Pi extension result deadline. |
| `omp` | `sessionDir` | non-empty `string` | `~/.omp/agent/sessions`; import-only OMP session directory. |
| `omp` | `rpcTimeoutMs` | positive integer | `60000`; also overrides OMP's default 20-second ready deadline. |
| `omp` | `smolModel` | non-empty `string` | Passed to OMP as `--smol`. |
| `omp` | `slowModel` | non-empty `string` | Passed to OMP as `--slow`. |
| `omp` | `planModel` | non-empty `string` | Passed to OMP as `--plan`. |

Pi and OMP param objects reject unknown keys when their adapters parse them. Generic ACP params preserve unknown keys but only the listed capabilities have core behavior. Plugin providers may define additional params.

`env` values are not expanded by Paseo. Avoid writing API keys directly. Arrange the daemon's environment or a trusted wrapper command where possible.

### Provider catalog timeout

| Key | Type | Default / behavior |
| --- | --- | --- |
| `agents.catalogRefreshTimeoutMs` | positive integer, max `2147483647` | Per-provider availability and catalog probe timeout. Runtime/provider fallback is used when omitted. |

The older `PASEO_PROVIDER_REFRESH_TIMEOUT_MS` environment fallback may apply when this field is absent, depending on the provider snapshot path.

### Metadata generation model order

`agents.metadataGeneration.providers` is an ordered array. Configured entries run before built-in candidates and the current agent/draft selection.

| Entry key | Type | Required | Meaning |
| --- | --- | --- | --- |
| `provider` | non-empty `string` | Yes | Exact built-in or custom provider ID. |
| `model` | non-empty `string` | No | Specific model; omit for the provider default. |
| `thinkingOptionId` | non-empty `string` | No | Requested reasoning option; falls back if unavailable. |

```json
{
  "agents": {
    "metadataGeneration": {
      "providers": [
        {
          "provider": "claude",
          "model": "claude-haiku-4-5-20251001",
          "thinkingOptionId": "low"
        },
        { "provider": "opencode" }
      ]
    }
  }
}
```

### Orchestration skill selection

`agents.skills.selection` accepts exactly one of:

```json
{ "mode": "all" }
```

```json
{ "mode": "custom", "skills": ["paseo", "paseo-handoff"] }
```

Missing selection resolves to all managed Paseo orchestration skills. Installed filesystem state is derived separately and is not stored in this object.

## Plugins

| Key | Type | Default / behavior |
| --- | --- | --- |
| `pluginsEnabled` | `boolean` | `false`; global switch for every configured plugin. |
| `plugins.<id>.source` | literal `"directory"` | Required. |
| `plugins.<id>.path` | non-empty `string` | Required directory path. |
| `plugins.<id>.enabled` | `boolean` | Enabled unless false. |

Plugin IDs match `^[a-z][a-z0-9-]*$`. Git-installed plugins still appear as directory sources here; `$PASEO_HOME/plugins/sources.json` owns Git origin and revision metadata.

Use plugin CLI operations instead of hand-editing source entries:

```bash
paseo plugin add owner/repository
paseo plugin install /absolute/path/to/plugin
paseo plugin ls
paseo plugin enable <id>
paseo plugin disable <id>
paseo plugin reload <id>
paseo plugin remove <id>
```

Plugins are unsandboxed trusted code. Never enable them without explicit user approval.

## `features`

### Dictation

| Key | Type | Default / behavior |
| --- | --- | --- |
| `features.dictation.enabled` | `boolean` | `true` when omitted. |
| `features.dictation.stt.provider` | `"local" | "openai"` | `local`. |
| `features.dictation.stt.model` | non-empty `string` | Local default `parakeet-tdt-0.6b-v2-int8`; OpenAI uses endpoint model resolution. |
| `features.dictation.stt.language` | trimmed non-empty `string` | `en`; applies to OpenAI STT. Local Parakeet v2 is English and v3 auto-detects. |
| `features.dictation.stt.confidenceThreshold` | `number` | OpenAI STT low-confidence threshold; runtime default `-3.0`. |

### Voice mode

| Key | Type | Default / behavior |
| --- | --- | --- |
| `features.voiceMode.enabled` | `boolean` | `true` when omitted. |
| `features.voiceMode.llm.provider` | `string` | Voice agent provider. Omission leaves automatic/no explicit provider behavior. |
| `features.voiceMode.llm.model` | non-empty `string` | Optional provider model. |
| `features.voiceMode.stt.provider` | `"local" | "openai"` | `local`. |
| `features.voiceMode.stt.model` | non-empty `string` | Local default `parakeet-tdt-0.6b-v2-int8`. |
| `features.voiceMode.stt.language` | trimmed non-empty `string` | Dictation language, then `en`; applies to OpenAI STT. |
| `features.voiceMode.turnDetection.provider` | `"local" | "openai"` | `local`; local uses bundled Silero VAD. |
| `features.voiceMode.tts.provider` | `"local" | "openai"` | `local`. |
| `features.voiceMode.tts.model` | non-empty `string` | Local default `kokoro-en-v0_19`; OpenAI default `tts-1`. |
| `features.voiceMode.tts.voice` | enum | OpenAI voice: `alloy`, `echo`, `fable`, `onyx`, `nova`, or `shimmer`; default `alloy`. |
| `features.voiceMode.tts.speakerId` | integer | Local TTS speaker; default `0`. |
| `features.voiceMode.tts.speed` | `number` | Local TTS speed. |

The persisted schema accepts `openai` for turn detection, but current practical turn detection is local; verify provider support before selecting otherwise.

### Bundled web UI

| Key | Type | Default / behavior |
| --- | --- | --- |
| `features.webUi.enabled` | `boolean` | `false` for normal daemon startup; official Docker startup may override it. |
| `features.webUi.distDir` | non-empty `string` | Bundled web UI directory. Relative paths resolve against `$PASEO_HOME`. |

Static UI files load without bearer auth. API and WebSocket requests still enforce daemon auth.

## `log`

Allowed levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`.

Allowed formats: `pretty`, `json`.

| Key | Type | Default / behavior |
| --- | --- | --- |
| `log.level` | log level | Legacy global level; retained for compatibility. |
| `log.format` | log format | Legacy global format; retained for compatibility. |
| `log.console.level` | log level | Console threshold; normal default `info`. |
| `log.console.format` | log format | Console format. |
| `log.file.level` | log level | File threshold; normal default `trace`. |
| `log.file.path` | non-empty `string` | `$PASEO_HOME/daemon.log`; relative paths resolve against `$PASEO_HOME`. |
| `log.file.rotate.maxSize` | non-empty `string` | `10m`; rotating-file size syntax. |
| `log.file.rotate.maxFiles` | positive integer | Supervisor source default is `3` total files. |

```json
{
  "log": {
    "console": { "level": "info", "format": "pretty" },
    "file": {
      "level": "trace",
      "path": "daemon.log",
      "rotate": { "maxSize": "10m", "maxFiles": 3 }
    }
  }
}
```

`PASEO_LOG_ROTATE_SIZE` and `PASEO_LOG_ROTATE_COUNT` are supervisor fallbacks only when persisted rotation fields are absent. The similarly named `PASEO_LOG_FILE_ROTATE_*` variables are daemon config overrides.

## Environment and CLI overrides

Environment and daemon-start flags beat persisted settings. Reload cannot replace a launch override.

### Daemon, relay, proxy, and UI

| Environment variable | Config path / behavior |
| --- | --- |
| `PASEO_HOME` | Changes config and state root. |
| `PASEO_LISTEN` | `daemon.listen` |
| `PORT` | Fallback port only when no listen address is set. |
| `PASEO_HOSTNAMES` | Appends to `daemon.hostnames`. |
| `PASEO_ALLOWED_HOSTS` | Deprecated hostname env alias. |
| `PASEO_TRUSTED_PROXIES` | `daemon.trustedProxies`; comma list, boolean true, or false for empty. |
| `PASEO_CORS_ORIGINS` | Comma list appended to `daemon.cors.allowedOrigins`. |
| `PASEO_PASSWORD` | Plaintext launch password; overrides persisted bcrypt hash. |
| `PASEO_APP_BASE_URL` | `app.baseUrl` |
| `PASEO_RELAY_ENABLED` | `daemon.relay.enabled` |
| `PASEO_RELAY_ENDPOINT` | `daemon.relay.endpoint` |
| `PASEO_RELAY_PUBLIC_ENDPOINT` | `daemon.relay.publicEndpoint` |
| `PASEO_RELAY_USE_TLS` | `daemon.relay.useTls` |
| `PASEO_RELAY_PUBLIC_USE_TLS` | `daemon.relay.publicUseTls` |
| `PASEO_SERVICE_PROXY_ENABLED` | Compatibility shim for optional proxy layers. |
| `PASEO_SERVICE_PROXY_LISTEN` | `daemon.serviceProxy.listen` |
| `PASEO_SERVICE_PROXY_PUBLIC_BASE_URL` | `daemon.serviceProxy.publicBaseUrl` |
| `PASEO_WEB_UI_ENABLED` | `features.webUi.enabled` |
| `PASEO_WEB_UI_DIST_DIR` | `features.webUi.distDir` |

Boolean env values accept common true/false forms (`1/0`, `true/false`, `yes/no`, `on/off`; speech flags also accept `y/n`).

CLI start overrides exist for listen/port, relay enabled/TLS, MCP enabled/injection, web UI enabled, hostnames, and home. Check `paseo daemon start --help` for the installed version.

### Git

| Environment variable | Config path |
| --- | --- |
| `PASEO_PROVIDER_REFRESH_TIMEOUT_MS` | Fallback for `agents.catalogRefreshTimeoutMs` when that field is absent. |
| `PASEO_GIT_MAX_PROCESSES_PER_SECOND` | `daemon.git.maxProcessesPerSecond` |
| `PASEO_GIT_MAX_PROCESS_CONCURRENCY` | `daemon.git.maxProcessConcurrency` |
| `PASEO_GIT_CONCURRENCY` | Deprecated concurrency alias; lower priority than the new name. |

### Logging

| Environment variable | Config path / behavior |
| --- | --- |
| `PASEO_LOG_LEVEL` | `log.level` |
| `PASEO_LOG` | Legacy `log.level` alias. |
| `PASEO_LOG_FORMAT` | `log.format` |
| `PASEO_LOG_CONSOLE_LEVEL` | `log.console.level` |
| `PASEO_LOG_CONSOLE_FORMAT` | `log.console.format` |
| `PASEO_LOG_FILE_LEVEL` | `log.file.level` |
| `PASEO_LOG_FILE_PATH` | `log.file.path` |
| `PASEO_LOG_FILE_ROTATE_SIZE` | `log.file.rotate.maxSize` |
| `PASEO_LOG_FILE_ROTATE_COUNT` | `log.file.rotate.maxFiles` |
| `PASEO_LOG_ROTATE_SIZE` | Supervisor rotation fallback when persisted max size is absent. |
| `PASEO_LOG_ROTATE_COUNT` | Supervisor rotation fallback when persisted max files is absent. |

### Speech and voice

| Environment variable | Config path / behavior |
| --- | --- |
| `PASEO_DICTATION_ENABLED` | `features.dictation.enabled` |
| `PASEO_DICTATION_STT_PROVIDER` | `features.dictation.stt.provider` |
| `PASEO_DICTATION_LOCAL_STT_MODEL` | Local `features.dictation.stt.model` |
| `PASEO_DICTATION_LANGUAGE` | `features.dictation.stt.language`; voice language fallback too. |
| `PASEO_VOICE_MODE_ENABLED` | `features.voiceMode.enabled` |
| `PASEO_VOICE_LLM_PROVIDER` | `features.voiceMode.llm.provider` |
| `PASEO_VOICE_STT_PROVIDER` | `features.voiceMode.stt.provider` |
| `PASEO_VOICE_LOCAL_STT_MODEL` | Local `features.voiceMode.stt.model` |
| `PASEO_VOICE_LANGUAGE` | `features.voiceMode.stt.language` |
| `PASEO_VOICE_TURN_DETECTION_PROVIDER` | `features.voiceMode.turnDetection.provider` |
| `PASEO_VOICE_TTS_PROVIDER` | `features.voiceMode.tts.provider` |
| `PASEO_VOICE_LOCAL_TTS_MODEL` | Local `features.voiceMode.tts.model` |
| `PASEO_VOICE_LOCAL_TTS_SPEAKER_ID` | `features.voiceMode.tts.speakerId` |
| `PASEO_VOICE_LOCAL_TTS_SPEED` | `features.voiceMode.tts.speed` |
| `PASEO_LOCAL_MODELS_DIR` | `providers.local.modelsDir` |
| `OPENAI_STT_API_KEY` | OpenAI STT credential. |
| `OPENAI_STT_BASE_URL` | OpenAI STT endpoint. |
| `OPENAI_TTS_API_KEY` | OpenAI TTS credential. |
| `OPENAI_TTS_BASE_URL` | OpenAI TTS endpoint. |
| `OPENAI_API_KEY` | Shared OpenAI speech credential fallback. |
| `OPENAI_BASE_URL` | Shared OpenAI speech endpoint fallback. |
| `STT_MODEL` | OpenAI STT model. |
| `STT_CONFIDENCE_THRESHOLD` | OpenAI STT threshold. |
| `TTS_MODEL` | OpenAI TTS model; accepted values are `tts-1` and `tts-1-hd`. |
| `TTS_VOICE` | OpenAI TTS voice enum. |

## Reload behavior

Run:

```bash
paseo reload
```

The daemon validates the complete file before applying it. A mixed edit can apply reloadable fields and report restart-required fields in the same result.

Reloadable paths:

- `daemon.relay.enabled`
- `daemon.mcp.enabled`
- `daemon.mcp.injectIntoAgents`
- `daemon.browserTools.enabled`
- `daemon.hostnames`
- `daemon.cors.allowedOrigins`
- `daemon.trustedProxies`
- `daemon.git.maxProcessesPerSecond`
- `daemon.git.maxProcessConcurrency`
- `daemon.autoArchiveAfterMerge`
- `daemon.enableTerminalAgentHooks`
- `daemon.appendSystemPrompt`
- `daemon.terminalProfiles`
- `daemon.agentProfiles`
- `app.baseUrl`
- `agents.providers`
- `agents.catalogRefreshTimeoutMs`
- `agents.metadataGeneration`
- `agents.skills.selection`
- `pluginsEnabled`

Other changed global paths require restart. Plugin source lifecycle should use `paseo plugin ...` commands.

## Validation and compatibility details

- Root, `daemon`, `app`, `worktrees`, `providers`, `agents`, `features`, and `log` objects are strict.
- `daemon.mcp`, `daemon.browserTools`, terminal profile entries, and agent profile entries are passthrough for wire compatibility.
- Provider `params` is intentionally arbitrary; adapter validation occurs later.
- Legacy provider entries with `command: { mode, ... }` may migrate on load. Write the current `command: string[]` form.
- Removed `providers.local.autoDownload` and `providers.openai.voice` fields are silently discarded before strict validation.
- A malformed global file stops config loading. Unknown strict keys are errors, not harmless extensions.
- The public generated schema may lag `PersistedConfigSchema`. Runtime source wins for the installed checkout.
