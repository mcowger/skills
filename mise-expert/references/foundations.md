# Mise foundations

Use this reference for installation, activation, configuration hierarchy, and the core mental model.

## Mental model

- `mise` manages runtimes and developer tools.
- It can automatically switch tool versions by directory.
- It usually determines active tools from `mise.toml` and other config files in the current directory hierarchy.
- It also understands `.tool-versions` and some idiomatic version files such as `.go-version`.

## Core commands

### `mise use`
Use when the user wants to both install a tool version and record it in config.

Examples:
- `mise use node@22`
- `mise use -g bun@latest`
- `mise use go@1.21`
- `mise use github:BurntSushi/ripgrep@latest`

Notes:
- By default, this writes to the local project config.
- `-g/--global` writes to global config.
- If multiple config files exist, mise writes to the lowest-precedence writable target such as `mise.toml` unless flags change that behavior.
- Use `--dry-run` when you want to preview changes.

### `mise install`
Use when the user wants installation without changing active config.

Examples:
- `mise install node@22`
- `mise install go@1.25`

Important distinction:
- `mise install` installs.
- `mise use` installs and records activation intent.

### `mise env`
Use for one-shot environment export, especially when permanent activation is not desired.

Examples:
- `eval "$(mise env -s bash)"`
- `mise env -J`
- `mise env --redacted`

### `mise set`
Use to write environment variables into mise-managed config.

Examples:
- `mise set NODE_ENV=development`
- `mise set -g EDITOR=nvim`

### `mise run`
Use to run tasks defined in `mise.toml` or task files.

## Installation and activation

### Installation
The simple install path from the docs is:
- `curl https://mise.run | sh`

By default, mise installs into `~/.local/bin` unless configured otherwise.

### Activation
Activation is how mise updates the shell environment so the right tools are on `PATH`.

Key rule:
- put `mise activate` in interactive shell rc files,
- do not treat login/profile files as the normal place for prompt-driven activation.

### PATH activation vs shims

#### PATH activation
- `mise activate <shell>` updates the shell environment when the prompt is displayed.
- This is the standard interactive-shell workflow.
- It supports more features than shims.

#### Shims
- `mise activate --shims` uses lightweight wrappers in the shims directory.
- Useful for some non-interactive or IDE contexts.
- Not a full replacement for all PATH-activation features.

## Config files and hierarchy

Primary project files include:
- `mise.toml`
- `mise.local.toml`
- `mise/config.toml`
- `.mise/config.toml`
- `.config/mise.toml`
- `.config/mise/config.toml`
- `.config/mise/conf.d/*.toml`

Important behavior:
- mise walks up the directory tree,
- collects config files,
- merges them,
- and lets more specific files override broader ones.

General guidance:
- `mise.toml` for shared project config,
- `mise.local.toml` for local-only overrides and secrets,
- global user config in `~/.config/mise/config.toml`.

## Common sections in `mise.toml`

- `[tools]`
- `[env]`
- `[settings]`
- `[tasks.<name>]`
- `[tool_alias]`
- `[shell_alias]`
- `[hooks]`

## Core advice patterns

- If the user wants automatic project switching, use `mise use` and activation.
- If they want an ephemeral run, use `mise exec` or `mise env`.
- If they are confused about "installed" vs "active", explain `install` vs `use`.
- If they are migrating from asdf, acknowledge `.tool-versions` compatibility but prefer native `mise.toml` going forward when appropriate.
