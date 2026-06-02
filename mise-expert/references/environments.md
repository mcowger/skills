# Environments, shell state, and related features

Use this reference for `[env]`, dotenv loading, required variables, shell aliases, hooks, and direnv interplay.

## `[env]`

Use `[env]` for environment variables that mise should manage.

Example:
```toml
[env]
NODE_ENV = 'development'
API_BASE = 'http://localhost:3000'
```

This is often preferable to ad hoc shell exports because it keeps project behavior tied to project config.

## Required variables

mise supports marking variables as required.

Example:
```toml
[env]
DATABASE_URL = { required = true }
API_KEY = { required = "Get your API key from the team vault" }
```

Use this when a project should fail clearly if a secret or required variable is missing.

## `env._` directives

Use `env._.*` for special environment behavior.

### `env._.file`
Loads environment variables from a dotenv-style file.

Preferred example from the docs:
```toml
[env]
_.file = '.env'
```

More explicit form when you need options:
```toml
[env]
_.file = { path = '.env', tools = true }
```

The docs specifically prefer `env._.file` / `_.file` over older deprecated top-level keys like `dotenv` or `env_file`.

### `env._.path`
Use to extend `PATH` in a structured way rather than manually rewriting PATH in shell rc files.

### `env._.source`
Use when a shell script must be sourced to produce environment variables.

Caveat:
- expensive sourced scripts can slow prompt-time activation because they may re-run often.

## Local vs shared environment data

General guidance:
- shared, non-secret environment defaults can live in `mise.toml`,
- secrets and machine-specific values should usually live in `mise.local.toml` or an external secret system,
- avoid committing secrets.

## `mise set`

Use `mise set` when the user wants a CLI-driven way to write env vars into config.

Examples:
- `mise set NODE_ENV=production`
- `mise set -g EDITOR=nvim`

The command can also target environment-specific config files.

## Shell aliases

Shell aliases are managed under `[shell_alias]`.

Example:
```toml
[shell_alias]
ll = 'ls -la'
gs = 'git status'
```

Behavior:
- aliases are set when entering a directory with that config,
- aliases are unset when leaving,
- child configs can override parent aliases.

Important distinction:
- shell aliases are shell conveniences,
- tool aliases change runtime/backend/version resolution.

## Hooks

Hooks are secondary but supported.

Examples:
```toml
[hooks]
cd = "echo 'changed directories'"
enter = "echo 'entered project'"
leave = "echo 'left project'"
preinstall = "echo 'about to install tools'"
postinstall = "echo 'installed tools'"
```

Important behavior:
- most hooks require `mise activate` to be active in the shell,
- `preinstall` and `postinstall` do not require shell activation in the same way,
- `postinstall` receives `MISE_INSTALLED_TOOLS` as JSON.

## direnv

The docs are explicit that direnv is not the preferred default with mise.

Important guidance:
- mise and direnv both manage environment state,
- conflicts often show up around `PATH`,
- simple coexistence can work when they manage different concerns,
- do not recommend direnv casually unless the user already depends on it.

If the user already uses direnv:
- explain the conflict clearly,
- prefer the simplest setup,
- avoid promising that all mise activation features will behave perfectly with direnv.

## When helping with env problems

Check these in order:
1. Which file defines the variable?
2. Is the value supposed to be shared or local/secret?
3. Is activation running at all?
4. Is a dotenv or sourced file involved?
5. Is direnv also modifying the environment?
6. Does `mise env` show the expected value?
