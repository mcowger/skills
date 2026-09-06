# OpenCode Permissions Reference

Source of truth: `packages/core/src/v1/config/permission.ts` and the permissions docs.

## Actions

Every rule resolves to one of:

- `"allow"` — run without approval
- `"ask"` — prompt the user for approval
- `"deny"` — block the action

Top-level shorthand sets every tool at once:

```jsonc
{ "permission": "allow" }
```

## Rule structure

```jsonc
{
  "permission": {
    "*": "ask",                 // any tool, scalar action
    "bash": "allow",            // a tool, scalar action
    "edit": {                   // a tool, granular object rules
      "*": "deny",
      "src/**": "allow"
    }
  }
}
```

A string value for `permission` or for a specific tool is shorthand for `{ "*": <action> }`.

### Evaluation order

- Object rules are evaluated by pattern match against the tool input, with the **last matching rule winning**.
- Convention: put the catch-all `"*"` rule **first**, more specific rules **after** it.
- Agent-level `permission` merges with global config; agent rules take precedence.
- `OPENCODE_PERMISSION` env var (JSON) merges on top of everything file-based.

### Patterns

- `*` — matches zero or more of any character
- `?` — matches exactly one character
- Everything else matches literally
- `~` or `$HOME` at the start expands to the home directory (mainly for `external_directory` and path-based rules like `edit`)

## Permission keys

| Key | Matches | Granular? |
| --- | --- | --- |
| `read` | File path | yes |
| `edit` | File path (covers `edit`, `write`, `patch`) | yes |
| `glob` | Glob pattern | yes |
| `grep` | Regex pattern | yes |
| `list` | Path | yes |
| `bash` | Parsed commands like `git status --porcelain` | yes |
| `task` | Subagent type | yes |
| `skill` | Skill name | yes |
| `lsp` | LSP queries | no |
| `question` | Questions to the user | no |
| `webfetch` | URL | yes |
| `websearch` | Search query | yes |
| `todowrite` | Todo updates | no |
| `external_directory` | Paths outside the working directory | yes |
| `doom_loop` | Same tool call repeating 3× with identical input | no |

## Defaults

When nothing is configured:

- Most tools default to `"allow"`
- `doom_loop` and `external_directory` default to `"ask"`
- `read` is `"allow"` but `.env` files are denied:

```jsonc
{
  "permission": {
    "read": { "*": "allow", "*.env": "deny", "*.env.*": "deny", "*.env.example": "allow" }
  }
}
```

## Auto mode

```bash
opencode --auto                    # TUI
opencode run --auto "Refactor it"  # headless
```

Auto mode approves any request that would otherwise ask; explicit `deny` rules still block. Toggle at runtime from the command palette (Enable/Disable auto-approve permissions). While active the prompt shows a muted `auto` indicator.

## The "ask" prompt

When approval is required, the UI offers:
- `once` — approve this request only
- `always` — approve matching patterns for the rest of the session (patterns suggested by the tool, e.g. a safe command prefix)
- `reject` — deny

## external_directory in depth

Any tool taking a path (`read`, `edit`, `glob`, `grep`, many `bash` commands) triggers `external_directory` when the path falls outside the working directory where OpenCode started.

```jsonc
{
  "permission": {
    "external_directory": { "~/workspace/**": "allow", "/tmp/**": "allow" },
    "edit": { "~/workspace/**": "deny" }
  }
}
```

- Allowed directories inherit the workspace defaults (`read` stays `allow`); layer extra rules to restrict specific tools inside them.
- `~`/`$HOME` expansion only rewrites the pattern — it does not make external paths part of the workspace.

## Common recipes

```jsonc
// Guard destructive git operations globally, allow in one agent
{
  "permission": { "bash": { "*": "allow", "git push *": "ask", "git commit *": "ask" } },
  "agent": {
    "ship": { "permission": { "bash": { "git push *": "allow" } } }
  }
}

// Read-only plan agent that may still edit docs
{
  "agent": {
    "plan": { "permission": { "edit": { "*": "deny", "**/*.md": "allow", "**/*.mdx": "allow" } } }
  }
}

// Kill a chatty subagent tool entirely
{ "permission": { "task": "deny" } }
```

## Legacy `tools` config

Since v1.1.1 `tools` is deprecated and merged into `permission` at load time:
- `tools: { "bash": false }` → `permission: { "bash": "deny" }`
- `tools: { "write": true }` → `permission: { "edit": "allow" }` (`write`/`edit`/`patch` all map to `edit`)
- Works for top-level, agent-level, and wildcard entries (`"mymcp_*": false` to disable all tools from an MCP server)
