# Tools and backends

Use this reference when the user is installing runtimes, adding CLI tools, or deciding which backend syntax to use.

## Registry and backend selection

The mise registry provides shorthand names for many tools, but users can also select explicit backends.

Examples:
- `mise use node@22`
- `mise use github:BurntSushi/ripgrep@latest`
- `mise use npm:prettier@latest`
- `mise use go:github.com/DarthSim/hivemind@latest`
- `mise use cargo:eza@latest`

General guidance from the docs:
- Prefer core tools, `aqua`, or `github` when they are a good fit.
- Language-specific package-manager backends like `npm`, `go`, and `cargo` have more prerequisites and more moving parts.
- Users can still explicitly choose those backends when they want them.

## Prefer these backend heuristics

### Core tools
Use the built-in/core runtime support when the user is installing common runtimes such as Go or Bun and does not need a custom backend.

### `github:` backend
Good for tools distributed as release assets from GitHub.

Example:
- `mise use -g github:BurntSushi/ripgrep`

Notes:
- Good default for prebuilt binaries.
- Supports asset autodetection.
- Can be customized with options like `asset_pattern` and `version_prefix` in `[tools]`.

Example config:
```toml
[tools]
"github:cli/cli" = { version = "latest", asset_pattern = "gh_*_linux_x64.tar.gz" }
```

### `npm:` backend
Good for npm-distributed CLIs.

Example:
- `mise use -g npm:prettier`

Requirements and caveats:
- Requires node/npm available.
- mise can use package managers like `aube`, `pnpm`, `bun`, or `npm` depending on settings.
- This is not the first choice when a stable prebuilt binary backend exists.

### `go:` backend
Good for Go CLIs that are normally installed with `go install`.

Example:
- `mise use -g go:github.com/DarthSim/hivemind`

Requirements and caveats:
- Requires `go` on `PATH`.
- Useful for Go-native CLI distribution.
- Can set tool options like `install_env` and `tags`.

Example config:
```toml
[tools]
"go:github.com/golang-migrate/migrate/v4/cmd/migrate" = { version = "latest", tags = "postgres" }
```

### `cargo:` backend
Good for Rust CLIs from crates.io or Git repos.

Examples:
- `mise use -g cargo:eza`
- `mise use cargo:https://github.com/username/demo@tag:v1.2.3`

Requirements and caveats:
- Requires cargo/rust.
- May compile from source depending on configuration and availability of prebuilt artifacts.
- The docs note interactions with `cargo-binstall` settings.

## Bun

For Bun itself, prefer the core plugin unless the user asks for something custom.

Examples:
- `mise use -g bun@latest`
- `mise use bun@1.2`

Important note from docs:
- Avoid `bun upgrade` outside mise because mise will not know about the change.

## Go

For the Go runtime itself, prefer the core plugin.

Examples:
- `mise use -g go@1.21`
- `mise use go@1.25`

Important notes:
- Older Go minor versions like 1.20 and below may require `prefix:` syntax in some cases.
- `.go-version` files are supported.
- For Go CLIs, the docs now prefer using the `go:` backend rather than legacy default-package files.

## Tool aliases vs shell aliases

Do not confuse these:

### `[tool_alias]`
Use to change backend mapping or define symbolic versions.

Examples:
```toml
[tool_alias]
node = 'github:company/custom-node'

[tool_alias.node.versions]
lts-iron = '20'
```

### `[shell_alias]`
Use for shell commands like `ll = "ls -la"`.

They solve different problems. If the user says "alias", determine which type they mean.

## When answering backend questions

Prefer to explain:
1. why a backend is appropriate,
2. what dependency it needs,
3. the lowest-friction alternative if there is one,
4. how the resulting `[tools]` entry will look.
