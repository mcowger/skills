# Codex Approvals, Sandbox, Permissions & Trust

Reference for `approval_policy`, `approvals_reviewer`, `sandbox_mode`, `[permissions]`, exec-policy, and `[projects]` trust. Sources: `ConfigToml` (`config_toml.rs`), `permissions_toml.rs`, protocol `AskForApproval` / `SandboxMode` / `TrustLevel`, `execpolicy` + `requirements_exec_policy` crates.

## Approval policy

```toml
approval_policy = "on-request"
approvals_reviewer = "user"

[auto_review]
policy = "Extra guardian instructions appended to the auto-review prompt."
```

| Value | Meaning |
| --- | --- |
| `untrusted` | Internal policy for untrusted projects: commands require approval unless an exec-policy rule explicitly allows them |
| `on-request` | **Default.** The model decides when to ask. `on-failure` is an accepted alias |
| `granular` | Fine-grained per-flow controls: each category is allowed (`true` → run) or auto-rejected (`false` → reject without prompting). Covers sandbox approvals (incl. `with_additional_permissions` / `require_escalated`) and exec-policy `prompt` rules |
| `never` | Never escalate to the user — failures return straight to the model |

CLI: `-a on-request | never` (`--ask-for-approval`), plus:
- `--approve-for-me` — sets `approvals_reviewer = "auto_review"` + `approval_policy = "on-request"` + `sandbox_mode = "workspace-write"` (guardian subagent reviews escalations in the workspace sandbox; rejections come back with a reason — continue safer, prove authorization, or ask the user)
- `--dangerously-bypass-approvals-and-sandbox` (alias `--yolo`) — no prompts, no sandbox. **Only in externally-sandboxed environments.**

`approvals_reviewer`: `user` (default) routes escalations to you; `auto_review` routes to the guardian reviewer. `requirements.toml` (`allowed_approval_policies`, `allowed_approvals_reviewers`) can constrain both.

## Sandbox modes

```toml
sandbox_mode = "workspace-write"

[sandbox_workspace_write]
writable_roots = ["/scratch", "/tmp/shared"]
network_access = false
exclude_tmpdir_env_var = true
exclude_slash_tmp = true
```

| Mode | Meaning |
| --- | --- |
| `read-only` | **Default.** No writes outside narrow allowances |
| `workspace-write` | Writes allowed under the workspace (+ `writable_roots`); network gated by `network_access` (restricted vs enabled) |
| `danger-full-access` | No sandboxing (pairs with yolo flags) |

Derivation order: `--sandbox` flag > `sandbox_mode` config > trust-implied default (trusted/untrusted trees without an explicit mode get `workspace-write`, except unsandboxed Windows → `read-only`), else `read-only`. On Windows without the experimental sandbox, `workspace-write` is downgraded to `read-only`. CLI: `-s read-only | workspace-write | danger-full-access`. Extra writable dirs per run: `--add-dir DIR` (repeatable); run elsewhere: `-C DIR`; isolate: `--worktree`.

Seatbelt (macOS) / Landlock (Linux) / Windows-sandbox backends enforce the mode; `codex sandbox` runs arbitrary commands under the host sandbox.

## Permission profiles

```toml
default_permissions = "my-team"   # ":name" = built-in; "name" = [permissions] table

[permissions.my-team]
description = "Team default."
extends = "base-profile"          # inheritance; child keys win; cycles are errors
# ... compiled into a runtime PermissionProfile (fs/network/tool rules)
```

`default_permissions` names starting with `:` resolve to built-ins. `[permissions.<name>]` profiles support `extends` chains (parents merged first, child overrides; undefined parents and cycles are errors). The effective profile compiles to filesystem/network/tool rules enforced alongside the sandbox. Requirements can force the sandbox mapping (`allowed_sandbox_modes`).

## Exec policy

Repo/user `.rules` files give command-level allow/prompt/deny beyond the coarse sandbox. `codex execpolicy check` validates files against a command; `codex exec --ignore-rules` skips user/project rule files. `requirements_exec_policy` lets admins pin policy from requirements layers.

## Project trust

```toml
[projects."/home/matt.cowger/workspace/myrepo"]
trust_level = "trusted"   # or "untrusted"
```

- Lookup order: normalized cwd → project root (detected via `project_root_markers`, default `[".git"]`, walking parents) → git repo root (resolves worktrees to the true root). Windows compares case-insensitively.
- **Trusted** → cwd/tree/repo config layers, hooks, and exec-policy load. **Untrusted** (or unknown) → those layers load *disabled* with a message pointing at the trust setting, and approvals fall back toward `untrusted` behavior.
- Trust is an input-loading guard, **not a sandbox** — it decides which config/code to ingest, not what commands may do.
- Denylisted project keys (silently dropped from project layers — set them in user config): `openai_base_url`, `chatgpt_base_url`, `apps_mcp_product_sku`, `responses_api_metadata`, `model_provider`, `model_providers`, `notify`, `profile`, `profiles`, `otel`, realtime overrides. Project layers also can't set `shell_snapshot` / `respect_system_proxy` / credential-broker proxy fields, sensitive `shell_environment_policy.set` entries, or TUI permission-mode remaps.

Manage trust interactively when prompted, or declare it in `~/.codex/config.toml` as above (this machine trusts `$HOME` and the active workspaces).
