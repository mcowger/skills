# Troubleshooting mise

Use this reference when the user says mise is broken, the wrong version is active, activation does not work, installs fail, or shell behavior is strange.

## First-line diagnostics

Start with the smallest useful checks:

- `mise --version`
- `mise doctor`
- `mise ls`
- `mise which <tool>`
- `which -a <tool>`
- `mise env`

These usually reveal whether the problem is:
- installation,
- config targeting,
- activation,
- PATH ordering,
- or backend-specific failure.

## Activation problems

### `mise activate` in the wrong shell file
The docs warn that `mise activate` should be used in interactive shell rc files, not profile/login files like:
- `~/.profile`
- `~/.bash_profile`
- `~/.zprofile`

Reason:
- prompt-driven activation depends on interactive shells.

If the user needs non-interactive behavior:
- consider shims,
- or use `mise exec` / `mise env` explicitly.

## Wrong tool version active

This is commonly a PATH or activation issue.

Recommended diagnostic sequence:
1. confirm the requested version in config,
2. run `mise ls <tool>`,
3. run `mise which <tool>` if available,
4. run `which -a <tool>`,
5. inspect whether a non-mise binary appears earlier on PATH,
6. inspect shell activation or shim setup.

Useful fixes:
- correct shell activation placement,
- ensure shim directory ordering is correct,
- use `MISE_ACTIVATE_AGGRESSIVE=1` only when justified and explain why.

## Install failures

If install output is unclear:
- use `mise install --raw <tool>@<version>` for direct backend IO,
- reduce parallelism if needed,
- inspect backend prerequisites.

Examples:
- `npm:` needs node/npm,
- `go:` needs go,
- `cargo:` needs cargo/rust.

## Debug signals

If ordinary checks are not enough, the docs suggest:
- `MISE_DEBUG=1`
- `MISE_TRACE=1`
- `MISE_LOG_FILE_LEVEL=debug`
- `MISE_LOG_FILE=/path/to/logfile`

Use these when you need evidence, not as a default first step.

## Slow shell prompts

The docs note that prompt slowness can come from:
- expensive `env._.source` scripts,
- many tools/plugins,
- network-dependent env directives.

Useful measurement commands include `MISE_TIMINGS=1` or `MISE_TIMINGS=2` with `mise hook-env`.

Explain the tradeoff:
- PATH activation pays cost on prompt display,
- shims pay cost on command invocation.

## New release not showing up

Possible causes:
- cached version metadata,
- external version-host lag.

Common fix:
- `mise cache clear`

But treat cache clearing as a targeted fix, not a reflex.

## Cleanup/reset options

These are stronger measures and should be explained before use:
- `mise self-update`
- `mise cache clear`
- `mise implode`

Guidance:
- prefer `mise doctor` and targeted diagnosis first,
- reserve `implode` for intentionally resetting mise state.

## Directory structure clues

Useful directories from the docs:
- config: `~/.config/mise`
- cache: `~/.cache/mise`
- state: `~/.local/state/mise`
- data/tools/plugins: `~/.local/share/mise`
- installs: `~/.local/share/mise/installs`
- shims: under the mise data directory

These are helpful when the user is debugging stale installs, plugin state, cache behavior, or PATH issues.
