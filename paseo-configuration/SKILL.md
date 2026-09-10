---
name: paseo-configuration
description: Configure and troubleshoot Paseo daemon settings, agent providers, profiles, tools, plugins, voice, logging, worktrees, workspace scripts, service ports, terminals, and metadata prompts through $PASEO_HOME/config.json and per-project paseo.json. TRIGGERS - paseo config, configure paseo, paseo settings, config.json, paseo.json, paseo daemon config, paseo provider, agent profile, terminal profile, paseo plugin config, paseo voice config, paseo worktree setup, paseo scripts, paseo service ports, paseo reload.
disable-model-invocation: true
---

# Paseo configuration

Configure Paseo's host-wide daemon and repository-specific workspace behavior. The source checkout for this reference is `~/workspace/paseo`.

## When to use this skill

Use this skill to:

- Edit or diagnose `$PASEO_HOME/config.json`, normally `~/.paseo/config.json`
- Edit a repository's lowercase `paseo.json`
- Configure daemon networking, relay, auth, MCP injection, browser tools, Git limits, or logging
- Define agent providers, custom binaries, models, Paseo tool policy, agent profiles, or terminal profiles
- Configure orchestration skill selection, metadata generation, plugins, dictation, voice mode, or the bundled web UI
- Change worktree location, setup and teardown commands, workspace scripts, services, terminals, or service-port allocation
- Decide whether a change can use `paseo reload` or needs `paseo daemon restart`

## Configuration files and authority

| Scope | Path | Format | Purpose |
| --- | --- | --- | --- |
| Global | `$PASEO_HOME/config.json` | Strict JSON | Daemon, providers, profiles, plugins, voice, logging, global worktree policy |
| Project | `<project-root>/paseo.json` | JSON | Worktree lifecycle, scripts/services, startup terminals, service ports, metadata wording |
| Editor schema | `https://paseo.sh/schemas/paseo.config.v1.json` | JSON Schema | Global config completion and validation |
| Runtime state | `$PASEO_HOME/**` | Mixed | Agents, workspaces, logs, credentials, plugin state; not configuration to hand-edit |

`PASEO_HOME` defaults to `~/.paseo`. `PASEO_HOME` or `paseo daemon start --home <path>` changes the global file location.

The lowercase filename matters: `paseo.json`, not `Paseo.json`.

### Source of truth

Use the runtime schemas when the checked-in website schema or prose disagrees:

- Global: `packages/server/src/server/persisted-config.ts` (`PersistedConfigSchema`)
- Global resolution and precedence: `packages/server/src/server/config.ts`
- Project: `packages/protocol/src/paseo-config-schema.ts`
- Project runtime behavior: `packages/server/src/utils/worktree.ts`

The generated website schema can lag new runtime fields. In the reviewed checkout it omits some accepted fields, including browser tools, profiles, skills, plugins, metadata generation, and global service-port allocation.

## Precedence and merge behavior

Global settings resolve in this order, lowest to highest:

1. Runtime defaults
2. `$PASEO_HOME/config.json`
3. Environment variables
4. Daemon-start CLI flags

Most higher-precedence values replace lower ones. Two important list settings append and deduplicate across persisted and environment sources:

- `daemon.hostnames`
- `daemon.cors.allowedOrigins`

`paseo.json` is a separate project policy file. It does not deep-merge with global `config.json`. The main overlap is service-port allocation: `worktree.servicePorts` replaces `worktrees.servicePorts` for that project, and an explicit service `port` wins over both.

## Safe editing workflow

1. Read the existing file. Do not replace unrelated settings.
2. Keep global config strict JSON: no comments or trailing commas.
3. Do not put plaintext credentials in a committed project `paseo.json`.
4. Validate the whole global file against the current Paseo runtime, not only the public schema.
5. Run `paseo reload` after a global edit.
6. Restart only when reload reports restart-required paths.

```bash
paseo reload
paseo daemon status
paseo daemon restart                 # only when reload says it is required
paseo daemon set-password            # preferred password setup; stores bcrypt
```

Project settings can be edited in the app's Project settings screen or directly in `paseo.json`. Paseo uses revision checks for app writes and reports a stale-write conflict instead of overwriting a newer disk edit.

## Global config quick start

```json
{
  "$schema": "https://paseo.sh/schemas/paseo.config.v1.json",
  "version": 1,
  "daemon": {
    "listen": "127.0.0.1:6767",
    "hostnames": ["localhost", ".localhost"],
    "trustedProxies": ["loopback"],
    "mcp": {
      "enabled": true,
      "injectIntoAgents": false
    },
    "browserTools": {
      "enabled": false
    },
    "git": {
      "maxProcessesPerSecond": 64,
      "maxProcessConcurrency": 8
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

Every field is optional. A new home is initialized with `version: 1`, local listen, the hosted app CORS origin, relay disabled, and the hosted app base URL.

The complete field catalog, defaults, validation, environment overrides, and reload behavior are in [references/global-config.md](references/global-config.md).

## Project config quick start

```json
{
  "worktree": {
    "setup": ["npm ci", "npm run build"],
    "teardown": "npm run db:drop || true",
    "terminals": [
      { "name": "logs", "command": "npm run logs" }
    ],
    "servicePorts": {
      "range": "3000-4000"
    }
  },
  "scripts": {
    "test": {
      "command": "npm test"
    },
    "web": {
      "type": "service",
      "command": "npm run dev -- --port $PASEO_PORT"
    }
  },
  "metadataGeneration": {
    "branchName": {
      "instructions": "Use <type>/<scope>-<description>."
    },
    "commitMessage": {
      "instructions": "Follow Conventional Commits."
    }
  }
}
```

The complete project schema and runtime semantics are in [references/project-config.md](references/project-config.md).

## Agent providers and profiles

Provider overrides live at `agents.providers.<id>`.

```json
{
  "agents": {
    "providers": {
      "codex": {
        "enabled": false
      },
      "my-claude": {
        "extends": "claude",
        "label": "Claude via gateway",
        "env": {
          "ANTHROPIC_AUTH_TOKEN": "set-this-in-the-daemon-environment",
          "ANTHROPIC_BASE_URL": "https://gateway.example.com/anthropic"
        },
        "disallowedTools": ["WebSearch"],
        "paseoTools": {
          "enabled": true,
          "disabledTools": ["browser_evaluate"]
        },
        "models": [
          {
            "id": "gateway-model",
            "label": "Gateway model",
            "isDefault": true
          }
        ]
      }
    }
  }
}
```

Rules:

- Built-in IDs are `claude`, `codex`, `copilot`, `opencode`, `pi`, and `omp`.
- Custom IDs match `^[a-z][a-z0-9-]*$` and require `extends` plus `label`.
- `extends` accepts a built-in ID or `acp`.
- ACP providers also require `command` and must speak ACP over stdio.
- `models` replaces the discovered catalog. `additionalModels` merges by model ID.
- `env` values are literal strings passed to the process. Paseo does not provide secret interpolation for global JSON.
- `params` is provider-specific. Do not invent keys; use the options in the global reference.
- `daemon.mcp.injectIntoAgents: false` disables the Paseo tool catalog globally. Per-provider `paseoTools` can narrow it further but cannot override the global disable.

Agent profiles under `daemon.agentProfiles` are named launch bundles. Terminal profiles under `daemon.terminalProfiles` are named commands. Both arrays replace the complete list when saved. Omitting terminal profiles restores built-ins; `[]` removes them all.

## Plugins

Plugin execution is globally off unless `pluginsEnabled` is `true`.

```json
{
  "pluginsEnabled": true,
  "plugins": {
    "review-tools": {
      "source": "directory",
      "path": "/absolute/path/to/plugin",
      "enabled": true
    }
  }
}
```

Plugins are trusted, unsandboxed code. Server code runs with daemon-user access and client contributions run inside Paseo. Never enable plugins for a user without explicit permission. Prefer `paseo plugin add|install|enable|disable|reload|remove` over manual lifecycle edits.

## Voice and speech

Speech provider selection is independent for dictation STT, voice STT, turn detection, and voice TTS. Each uses `local` or `openai`; local is the default. Voice LLM selection reuses an agent provider.

```json
{
  "features": {
    "dictation": {
      "enabled": true,
      "stt": {
        "provider": "local",
        "model": "parakeet-tdt-0.6b-v2-int8"
      }
    },
    "voiceMode": {
      "enabled": true,
      "llm": { "provider": "claude", "model": "haiku" },
      "stt": { "provider": "local" },
      "turnDetection": { "provider": "local" },
      "tts": {
        "provider": "local",
        "model": "kokoro-en-v0_19",
        "speakerId": 0,
        "speed": 1
      }
    }
  },
  "providers": {
    "local": {
      "modelsDir": "~/.paseo/models/local-speech"
    }
  }
}
```

Use `providers.openai.stt` and `.tts` for separate endpoints. Do not commit API keys; environment variables are safer.

## Worktree scripts and services

- `worktree.setup` runs after Paseo creates a worktree.
- `worktree.teardown` runs before archive removes it.
- Plain scripts run on demand and finish.
- `type: "service"` scripts are supervised and receive an allocated port and proxy URL.
- Bind services to `$PASEO_PORT`; `PORT` is not injected automatically.
- `portScript` runs directly without a shell. It must be an executable path, not an inline command.
- Lifecycle commands use non-login Bash on macOS/Linux and PowerShell on Windows. Use a Node script for cross-platform lifecycle logic.

## Reload versus restart

Common runtime-safe fields:

- Relay enabled state
- MCP and browser-tool switches
- Hostnames, CORS, trusted proxies, and Git limits
- Auto-archive, terminal hooks, appended prompt, profiles, agent providers, and metadata provider order
- App base URL, skill selection, provider catalog timeout, and `pluginsEnabled`

Common restart-required fields:

- Listen and password auth
- Relay endpoints and TLS
- Worktree root and global service-port allocation
- Service-proxy addresses
- Bundled web UI
- Logging
- Speech, voice, credentials, and local model directory
- Plugin source entries (use plugin lifecycle commands instead)

Always trust the paths reported by `paseo reload`; launch environment and CLI overrides can keep file changes from taking effect.

## Security rules

- `daemon.auth.password` stores a bcrypt hash, not plaintext. Use `paseo daemon set-password` or `PASEO_PASSWORD`.
- `providers.openai.*.apiKey` and `agents.providers.*.env` can contain secrets. Keep `$PASEO_HOME/config.json` private and prefer process environment where supported.
- A committed `paseo.json` can execute setup, teardown, scripts, services, and port allocators. Review it as executable project code.
- `daemon.hostnames: true` accepts any Host header. Keep the default allowlist unless broad access is intended.
- `daemon.trustedProxies: true` trusts forwarded headers from every peer. Use explicit proxy names or CIDRs unless the final proxy overwrites client-supplied headers.
- Browser tools can control logged-in tabs. Enable them only for trusted agents.
- Paseo tool filtering controls catalog exposure, not OS authorization. An agent with shell access may still reach the host directly.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Daemon rejects the whole global file | Unknown key, invalid JSON, wrong type, or failed strict validation | Compare with [references/global-config.md](references/global-config.md); remove comments/trailing commas |
| Editor rejects a field that runtime accepts | Generated public schema lags the runtime Zod schema | Check the current runtime schema; do not delete a valid newer field only to satisfy stale completion |
| Global change has no effect after reload | Field needs restart or a launch env/CLI override controls it | Read `restartRequiredPaths` and `overrideControlledPaths`; remove the override and restart |
| Project config is ignored | Wrong case/location, invalid JSON, or Paseo is reading the committed base-branch copy | Use lowercase `<project-root>/paseo.json`; commit the relevant version for worktree creation |
| Service has no URL | Script is not exactly `type: "service"`, command is empty, or startup failed | Fix the entry and inspect the supervised terminal/script status |
| Service cannot bind | Hard-coded port, exhausted configured range, invalid allocator output, or collision | Bind `$PASEO_PORT`; verify range or `portScript`; explicit `port` wins |
| Custom provider is rejected | Invalid ID, missing `extends`/`label`, invalid base, or ACP missing `command` | Follow the provider validation rules above |
| Provider model list is wrong | `models` replaced discovery | Use `additionalModels` to add or relabel while keeping discovered models |
| Paseo/browser tools are absent | Global MCP injection off, provider policy disabled, browser tools off, or no desktop browser host | Check all layers; new/reloaded sessions pick up the policy |
| Terminal defaults disappeared | `daemon.terminalProfiles` is `[]` | Remove the field to restore built-ins |
| Plugin is configured but inactive | `pluginsEnabled` false or plugin entry `enabled: false` | Confirm trust, then enable globally and per entry |
| OpenAI voice has no credentials | Endpoint-specific and fallback keys all absent | Set `OPENAI_STT_API_KEY` / `OPENAI_TTS_API_KEY` or the appropriate persisted key |
