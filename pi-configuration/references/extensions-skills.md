# Pi Extensions, Packages, Skills & Prompts Reference

## Extensions

Discovery:
- Global `~/.pi/agent/extensions/*.ts` and `*/index.ts`
- Project `.pi/extensions/*.ts` and `*/index.ts` (requires trust)
- `settings.extensions` paths (relative to config dir)
- Temporary: `pi -e ./extension.ts`, `pi -e npm:@foo/bar`
- Disable: `--no-extensions` (explicit `-e` still loads)

Extensions run with the full OS permissions of the Pi process — treat them as trusted code.

### Extension API (configuration-relevant)

Registration:
`pi.registerTool()` `pi.registerCommand()` `pi.registerShortcut()` `pi.registerFlag()`
`pi.registerProvider()` / `pi.unregisterProvider()`
`pi.registerMessageRenderer()` `pi.registerEntryRenderer()` `pi.registerMarkdownTransformer()`
`pi.setActiveTools()` `pi.sendMessage()` `pi.sendUserMessage()` `pi.appendEntry()`

Events:
`project_trust` `resources_discover` `tool_call` `tool_result` `before_agent_start`
`before_provider_headers` `before_provider_request` `after_provider_response`
`session_before_compact` `context`

Extensions can add: tools, commands, shortcuts, CLI flags, providers, OAuth flows,
streaming APIs, permission gates, path protections, custom compaction, system-prompt edits,
custom UI/widgets/headers/footers/editors, resource discovery, session persistence.

### Permission gate example (`examples/extensions/permission-gate.ts` in pi repo)

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.toolName !== "bash") return;
  if (event.input.command.includes("rm -rf")) {
    if (!ctx.hasUI) return { block: true, reason: "Blocked (no UI)" };
    const ok = await ctx.ui.confirm("Dangerous command", "Allow?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

## Packages

Packages bundle extensions/skills/prompts/themes.

```bash
pi install npm:@foo/bar@1.0.0     # npm (version pin supported)
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo
pi install ./local-package
pi install -l npm:@foo/bar        # project-local (.pi/npm/, .pi/git/)
pi remove npm:@foo/bar
pi list
pi update [--all|--extensions|--models|--self [pkg]]
pi config [-l]                    # edit package/resource config (-l = project)
```

Global installs: `~/.pi/agent/npm/`, `~/.pi/agent/git/`.

Manifest (package.json):
```json
{ "name": "my-pi-package", "keywords": ["pi-package"],
  "pi": { "extensions": ["./extensions"], "skills": ["./skills"],
          "prompts": ["./prompts"], "themes": ["./themes"] } }
```
Without a `pi` block, conventional `extensions/ skills/ prompts/ themes/` dirs are used.

#### Resource filtering (settings `packages` object form)

```json
{ "packages": [ { "source": "npm:my-package",
  "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
  "skills": ["-skills/mcp-scripting/SKILL.md"],
  "prompts": ["prompts/review.md"],
  "themes": ["+themes/legacy.json"] } ] }
```

- omitted field: load all of that type; `[]`: none
- `!pattern` exclude; `+path` / `-path` force include/exclude exact path
- `autoload: false`: start empty, apply only explicit patterns

Scope & dedup: npm identity = package name; git = repo URL without ref; local = resolved path.
A project entry normally wins over global; with `autoload: false` it acts as a delta.

Packaging rules: runtime deps in `dependencies` (not devDependencies); list bundled pi
deps as peerDependencies: `@earendil-works/pi-ai`, `@earendil-works/pi-agent-core`,
`@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, `typebox`.

## Skills

Discovery paths:
- Global: `~/.pi/agent/skills/`, `~/.agents/skills/`
- Project: `.pi/skills/`, `.agents/skills/` (cwd and ancestors)
- Packages: `<package>/skills/`
- `settings.skills` extra paths; `pi --skill ./path/SKILL.md`; `--no-skills`

Rules:
- Directories containing `SKILL.md` discovered recursively
- `.pi/skills/` and `~/.pi/agent/skills/` also allow root-level `.md` skills
- `.agents/skills/` requires nested dirs (root `.md` ignored)
- Missing/malformed `description` prevents loading; first wins on name collision
- Explicit `--skill` works with `--no-skills`

Frontmatter (name, description required):

```markdown
---
name: my-skill                 # 1–64 chars, lowercase + digits + hyphens
description: What it does and when to use it.   # ≤ 1024 chars
license: MIT
compatibility: Requires Node.js and jq.         # ≤ 500 chars
metadata: { category: documentation }
allowed-tools: read bash                        # experimental, space-delimited
disable-model-invocation: false                 # true = hidden from system prompt
---

# My Skill
Instructions go here.
```

Invocation (when `enableSkillCommands: true`, the default): `/skill:my-skill [args]`
— args append to the skill content as user instructions.

## Prompt templates

Discovery (non-recursive unless configured): `~/.pi/agent/prompts/*.md`,
`.pi/prompts/*.md`, package `prompts/`, `settings.prompts`, `pi --prompt-template`,
`--no-prompt-templates`.

Filename becomes the slash command (`review.md` → `/review`). Optional frontmatter
`description` (else first non-empty line) and `argument-hint`.

```markdown
---
description: Review staged git changes
argument-hint: "[focus]"
---
Review the staged changes for bugs, security issues, and error handling gaps.
Focus on: ${1:-all areas}
```

Substitutions: `$1`, `$2`, ..., `$@`, `$ARGUMENTS`, `${1:-default}`, `${@:-default}`,
`${@:N}`, `${@:N:L}` (slice).

## Context files

`AGENTS.md` (global: `~/.pi/agent/`; cwd + ancestors for project rules),
`SYSTEM.md` / `APPEND_SYSTEM.md` (global or `.pi/`).
CLI: `--system-prompt`, `--append-system-prompt`, `--no-context-files`.
