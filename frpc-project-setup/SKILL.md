---
name: frpc-project-setup
description: Install and configure frpc for project development tunnels, including mise setup, environment variables, deterministic worktree subdomains, dev-server lifecycle integration, and troubleshooting.
disable-model-invocation: true
---

# frpc project setup

Use this as the implementation guide when adding optional FRP HTTP tunnels to a
development server. The local app should still run normally when frpc is not
installed or FRP credentials are missing.

## What gets installed

`frpc` is the FRP client. It connects to an `frps` server and publishes a local
HTTP service.

For mise-managed projects, add this to the project `mise.toml`:

```toml
[tools]
"github:fatedier/frp" = "latest"
```

Then run:

```bash
mise install
frpc --version
```

Without mise, install the FRP release for the host OS and put the `frpc`
executable on `PATH`. Confirm it with `frpc --version` before testing the
project.

Do not commit FRP tokens. Keep them in the shell environment, a local ignored
`.env` file, or the project's secret manager.

## Environment contract

The project dev server should support these variables:

```bash
FRPC_SERVER_ADDR=your-frps-host
FRPC_AUTH_TOKEN=your-frps-token
FRPC_SERVER_PORT=7000
FRPC_SUBDOMAIN_HOST=dev.example.com
```

- `FRPC_SERVER_ADDR` is the `frps` control-plane address.
- `FRPC_AUTH_TOKEN` authenticates the client.
- `FRPC_SERVER_PORT` is the `frps` control port. Default it to `7000` and
  validate it as an integer from `1` through `65535`.
- `FRPC_SUBDOMAIN_HOST` is optional display metadata. It can be used to print
  an HTTPS URL, but it does not configure the `frps` routing suffix.
- `PORT` remains the local application's port. Pass it to frpc as the local
  port; do not confuse it with `FRPC_SERVER_PORT`.

If neither required credential is set, log that the tunnel is disabled and
continue. If only one is set, log a warning and continue without a tunnel.

## Client command

Use the CLI form so the project does not need to generate a temporary frpc
config file:

```bash
frpc http \
  --server-addr "$FRPC_SERVER_ADDR" \
  --server-port "$FRPC_SERVER_PORT" \
  --token "$FRPC_AUTH_TOKEN" \
  --proxy-name "$SUBDOMAIN" \
  --local-ip 127.0.0.1 \
  --local-port "$PORT" \
  --sd "$SUBDOMAIN"
```

Build the argument array in code instead of interpolating credentials into a
shell command. Spawn `frpc` with the project working directory and inherited
environment.

## Stable worktree names

Each worktree needs a stable, DNS-safe subdomain so restarts use the same URL
and parallel worktrees do not collide.

Derive it from:

```text
<repository-name>-<worktree-directory-name>
```

Normalize both values by lowercasing, replacing runs of non-`a-z0-9`
characters with `-`, and trimming leading and trailing `-` characters. Use
`repo` and `worktree` as fallbacks for empty values.

DNS labels have a 63-character limit. For longer values, keep a shortened
prefix and append `-` plus the first eight hexadecimal characters of the
SHA-256 hash of the original repository and worktree names separated by a NUL.

Extract the repository name from `remote.origin.url`, supporting both HTTPS
and SSH remotes. If Git metadata is unavailable, fall back to the current
directory basename.

The proxy name and subdomain should be the same unless the FRP deployment has a
specific naming requirement.

## Dev-server lifecycle

Start the tunnel only after the local service passes its health check. This
avoids publishing a route to a process that has not finished booting.

Recommended sequence:

1. Select or derive the local `PORT`.
2. Start the development server.
3. Poll the real health endpoint until it is ready.
4. Check `frpc --version`, credentials, and `FRPC_SERVER_PORT`.
5. Derive the endpoint and spawn frpc.
6. Log the subdomain, or the full URL when `FRPC_SUBDOMAIN_HOST` is set.

Track the frpc child separately from the application process. On `SIGINT`,
`SIGTERM`, `SIGHUP`, restart, and startup failure, terminate frpc before
returning. Do not make an unavailable FRP server prevent local development.

For detached dev servers, use a PID file and log file. The stop command must
terminate the complete application process group and the frpc child, then
remove the PID file. Avoid leaving a tunnel connected after the local service
has stopped.

## URL helper

Provide a command that derives the current worktree endpoint without scraping
logs, for example:

```bash
bun run dev:get:frp-url
bun run dev:get:frp-url -- --hostname
bun run dev:get:frp-url -- --subdomain
bun run dev:get:frp-url -- --json
```

The helper should use the exact same endpoint function as the dev lifecycle.
It derives the route; it does not prove that the public tunnel is reachable.

If no `FRPC_SUBDOMAIN_HOST` is configured, require `--subdomain` or `--json`
instead of printing a misleading full URL.

## Validation checklist

Run these checks after setup:

```bash
frpc --version
FRPC_SERVER_ADDR=... FRPC_AUTH_TOKEN=... bun run dev
bun run dev:get:frp-url -- --json
```

Confirm that:

- the local health endpoint becomes ready before frpc starts;
- the logged local port matches the frpc `--local-port`;
- the subdomain stays the same across restarts;
- separate worktrees produce separate subdomains;
- long names remain valid DNS labels;
- missing credentials or a missing binary disable only the tunnel;
- stopping the dev server stops frpc too.

Add unit tests for remote parsing, DNS normalization, deterministic endpoint
generation, long-label hashing, URL formatting, and the exact argument array.
