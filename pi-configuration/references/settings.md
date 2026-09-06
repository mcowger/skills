# Pi Settings Reference

Complete catalog for `~/.pi/agent/settings.json` (global) and `.pi/settings.json` (project).
Strict JSON (no comments). Project files require project trust. Nested objects deep-merge;
arrays and scalars replace wholesale.

## Paths, sessions & trust

- Global root: `~/.pi/agent/` (override `PI_CODING_AGENT_DIR`)
- Project root: `.pi/` (declared via package manifest `"piConfig": { "configDir": ".pi" }`)
- Sessions: precedence `--session-dir` > `PI_CODING_AGENT_SESSION_DIR` > `settings.sessionDir` > `~/.pi/agent/sessions/`
- Trust storage: `~/.pi/agent/trust.json` (current dir, inherits closest trusted parent)
- Settings paths resolve relative to the containing config dir; `~`/absolute allowed.

## Model & thinking

| Key | Default | Meaning |
|---|---|---|
| `defaultProvider` | unset | Startup provider (e.g. `anthropic`, `openai`, `plexus`) |
| `defaultModel` | unset | Startup model ID |
| `defaultThinkingLevel` | unset | `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max` |
| `modelThinkingLevels` | unset | Per-model levels keyed `provider/modelId` |
| `thinkingBudgets` | unset | Custom token budgets per level, e.g. `{"high": 32768}` |
| `enabledModels` | unset | Models available for Ctrl+P cycling (globs: `claude-*`) |
| `hideThinkingBlock` | `false` | Hide thinking blocks in output |
| `showCacheMissNotices` | `false` | Cache-miss/compaction/branch/provider-recovery notices |

## UI & display

| Key | Default | Meaning |
|---|---|---|
| `theme` | `"dark"` | Theme name (built-ins `dark`, `light`) |
| `externalEditor` | `$VISUAL`/`$EDITOR` | Ctrl+G editor command; beats env vars |
| `quietStartup` | `false` | Hide startup header |
| `collapseChangelog` | `false` | Condense post-update changelog |
| `enableInstallTelemetry` | `true` | Anonymous install ping |
| `enableAnalytics` | `false` | Opt-in analytics |
| `trackingId` | unset | Analytics identifier |
| `defaultProjectTrust` | `"ask"` | `ask`, `always`, `never` |
| `doubleEscapeAction` | `"tree"` | `tree`, `fork`, `none` |
| `treeFilterMode` | `"default"` | Initial `/tree` filter |
| `editorPaddingX` | `0` | Editor horizontal padding (0–3) |
| `outputPad` | `1` | Chat output padding (0–1) |
| `autocompleteMaxVisible` | `5` | Autocomplete rows (3–20) |
| `showHardwareCursor` | `false` | Terminal cursor during IME positioning |
| `tuiMode` | `"regular"` | `regular` or experimental `fullscreen` |
| `fullscreenExitOutput` | `"transcript"` | `transcript` or `resume-hint` |
| `fullscreenScrollbar` | `"auto"` | `auto`, `always`, `hidden` |
| `fullscreenCopyOnSelect` | `true` | Auto-copy fullscreen selections |
| `markdown.codeBlockIndent` | `"  "` | Code block indent |
| `markdown.mermaid` | `"streaming"` | `off`, `final`, `streaming` |

## Network

| Key | Default | Meaning |
|---|---|---|
| `httpProxy` | unset | Applied as HTTP(S)_PROXY (global only) |
| `transport` | `"auto"` | `sse`, `websocket`, `websocket-cached`, `auto` |
| `httpIdleTimeoutMs` | `300000` | `0` disables |
| `websocketConnectTimeoutMs` | `15000` | `0` disables |

## Warnings / message delivery

| Key | Default | Meaning |
|---|---|---|
| `warnings.anthropicExtraUsage` | `true` | Warn about Anthropic subscription paid extra usage |
| `steeringMode` | `"one-at-a-time"` | `all` or `one-at-a-time` |
| `followUpMode` | `"one-at-a-time"` | `all` or `one-at-a-time` |

## Compaction & branch summaries

```json
{ "compaction": { "enabled": true, "reserveTokens": 16384, "keepRecentTokens": 20000 } }
```

| Key | Default | Meaning |
|---|---|---|
| `compaction.enabled` | `true` | Auto-compaction |
| `compaction.reserveTokens` | `16384` | Response token reserve |
| `compaction.keepRecentTokens` | `20000` | Recent tokens kept verbatim |
| `branchSummary.reserveTokens` | `16384` | Branch summarization reserve |
| `branchSummary.skipPrompt` | `false` | Skip `/tree` "Summarize branch?" prompt |

## Retry

```json
{ "retry": { "enabled": true, "maxRetries": 3, "baseDelayMs": 2000,
             "provider": { "timeoutMs": 3600000, "maxRetries": 0, "maxRetryDelayMs": 60000 } } }
```

Agent-level retries for transient errors + `provider.*` for request timeout/retries.
`maxRetryDelayMs: 0` removes the cap on server-requested waits.

## Terminal & images

```json
{ "terminal": { "showImages": true, "imageWidthCells": 60, "clearOnShrink": false,
               "hyperlinks": "auto", "images": "auto", "trueColor": "auto",
               "showTerminalProgress": false },
  "images": { "autoResize": true, "blockImages": false } }
```

- `terminal.images`: `kitty`, `iterm2`, `false`, `auto`. `hyperlinks`/`trueColor`: bool or `"auto"`.
- `terminal.showTerminalProgress`: OSC 9;4 progress indicators.
- `images.autoResize`: cap at 2000×2000; `images.blockImages`: never send images to providers.

## Shell & packages

| Key | Meaning |
|---|---|
| `shellPath` | Custom shell for bash tool |
| `shellCommandPrefix` | Prefix prepended to every bash command |
| `npmCommand` | argv replacement for npm (e.g. `["mise","exec","node@20","--","npm"]`) |
| `sessionDir` | Custom session dir |

## Tools

- `defaultTools`: initial built-in tools from `read bash powershell edit write grep find ls`; `[]` disables built-ins but keeps extension/SDK tools. Project array replaces global.
- CLI: `--tools` strict allowlist (all tools), `--exclude-tools` filter, `--no-builtin-tools`, `--no-tools`.

## Resources

| Key | Meaning |
|---|---|
| `packages` | npm/git/local package sources (strings or filter objects) |
| `extensions` | Extra extension files/dirs |
| `skills` | Extra skill files/dirs |
| `prompts` | Extra prompt template files/dirs |
| `themes` | Extra theme files/dirs |
| `enableSkillCommands` | `true` (default): register `/skill:<name>` commands |

Resource arrays support globs, `!pattern` exclusion, `+path`/`-path` exact force include/exclude.

## Environment variables (full list)

### Process config

| Variable | Effect |
|---|---|
| `PI_CODING_AGENT_DIR` | Global config dir override |
| `PI_CODING_AGENT_SESSION_DIR` | Session dir override |
| `PI_PACKAGE_DIR` | Installed package dir override |
| `PI_OFFLINE` | No startup network ops / update checks / telemetry |
| `PI_SKIP_VERSION_CHECK` | Skip latest-version request |
| `PI_TELEMETRY` | `1/true/yes` or `0/false/no` |
| `PI_CACHE_RETENTION=long` | Extended provider prompt caching |
| `PI_SHARE_VIEWER_URL` | Base URL for `/share` |
| `PI_HARDWARE_CURSOR=1` | Hardware cursor |
| `PI_HYPERLINKS` | `1`, `0`, `auto` |
| `PI_IMAGE_PROTOCOL` | `kitty`, `iterm2`, `none`, `auto` |
| `PI_TRUE_COLOR` | `1`, `0`, `auto` |
| `PI_TUI_ESC_TIMEOUT` | Escape-sequence timeout (ms) |
| `PI_CLEAR_ON_SHRINK=1` | Clear terminal rows when content shrinks |
| `PI_TUI_WRITE_LOG` | Raw ANSI TUI output log path |
| `VISUAL` / `EDITOR` | External editor fallback (below `externalEditor` setting) |
| `HTTP_PROXY` / `HTTPS_PROXY` | Proxies |
| `PI_EXPERIMENTAL` | Experimental features (examples/source workflows) |

### Process markers (inherited by children)

`AI_AGENT=pi`, `PI_CODING_AGENT=true`.

### Shell-tool session variables

Injected into built-in bash/powershell tool runs only (not user `!`/`!!` commands):
`PI_SESSION_ID`, `PI_SESSION_FILE` (unset for ephemeral), `PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL`.

### Provider API keys (built-in mapping)

| Provider | Env var | `auth.json` key |
|---|---|---|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` | `azure-openai-responses` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| NVIDIA NIM | `NVIDIA_API_KEY` | `nvidia` |
| Amazon Bedrock | `AWS_BEARER_TOKEN_BEDROCK` | `amazon-bedrock` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| Cerebras | `CEREBRAS_API_KEY` | `cerebras` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway` |
| ZAI / ZAI China | `ZAI_API_KEY` / `ZAI_CODING_CN_API_KEY` | `zai` / `zai-coding-cn` |
| OpenCode / Go | `OPENCODE_API_KEY` | `opencode` / `opencode-go` |
| Radius | `RADIUS_API_KEY` | `radius` |
| Hugging Face | `HF_TOKEN` | `huggingface` |
| Fireworks | `FIREWORKS_API_KEY` | `fireworks` |
| Together AI | `TOGETHER_API_KEY` | `together` |
| Baseten | `BASETEN_API_KEY` | `baseten` |
| Kimi For Coding | `KIMI_API_KEY` | `kimi-coding` |
| MiniMax / China | `MINIMAX_API_KEY` / `MINIMAX_CN_API_KEY` | `minimax` / `minimax-cn` |
| Qwen Token Plan / China | `QWEN_TOKEN_PLAN_API_KEY` / `_CN_` | `qwen-token-plan` / `qwen-token-plan-cn` |
| Xiaomi | `XIAOMI_API_KEY` | `xiaomi` |
| Ant Ling | `ANT_LING_API_KEY` | `ant-ling` |

Cloud config extras: `AZURE_OPENAI_BASE_URL|RESOURCE_NAME|API_VERSION|DEPLOYMENT_NAME_MAP`,
`AWS_PROFILE|ACCESS_KEY_ID|SECRET_ACCESS_KEY|REGION`, `AWS_ENDPOINT_URL_BEDROCK_RUNTIME`,
`AWS_BEDROCK_SKIP_AUTH|FORCE_HTTP1|FORCE_CACHE`, `CLOUDFLARE_ACCOUNT_ID|GATEWAY_ID`,
`GOOGLE_CLOUD_PROJECT|LOCATION|APPLICATION_CREDENTIALS`.
