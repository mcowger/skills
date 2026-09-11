# OMP Settings Catalog & Reference

This reference documents the settings schema, types, default values, and behaviors for Oh My Pi (`config.yml`). Defaults and enum values come from `packages/coding-agent/src/config/settings-schema.ts`; `omp config list` is the authoritative live reference.

## Quick CLI Reference

```bash
omp config list                        # List all effective settings with types
omp config list --json                 # Export as JSON { key: { value, type, description } }
omp config get <key>                   # Get single key value (e.g. omp config get theme.dark)
omp config set <key> <value>           # Set single key value (writes global config.yml)
omp config reset <key>                 # Write the schema DEFAULT back (does not delete the key)
omp config path                        # Output path to active agent directory
omp config init-xdg                    # Create omp dirs under XDG data/state/cache homes (Linux/macOS)
```

`--json` is accepted by `list`, `get`, `set`, and `reset`. Unknown keys exit non-zero. In `list`, credential fields are masked (`********`); `get` returns them unmasked.

### Value parsing (`omp config set`)

| Type    | Accepted input                                      |
| ------- | --------------------------------------------------- |
| boolean | `true`, `false`, `yes`, `no`, `on`, `off`, `1`, `0` |
| number  | Any finite number (`Infinity`/`NaN` rejected)       |
| enum    | One of the key's allowed values, exact match        |
| array   | A JSON array, e.g. `'["anthropic","openai"]'`       |
| record  | A JSON object, e.g. `'{"bash":"prompt"}'`           |
| string  | Stored trimmed; multi-word values joined with spaces |

Keys must match a schema path exactly — no shorthand. Use `theme.dark`, not `theme`.

---

## 1. Models & Roles (`modelRoles`, `modelTags`, `cycleOrder`)

Built-in roles: `default`, `smol`, `slow`, `vision`, `plan`, `commit`, `tiny`, `task`, `advisor`. Role values may carry a thinking suffix (`:minimal`, `:low`, `:medium`, `:high`, `:xhigh`, `:max`). Extra roles can be introduced via `modelTags` or by assigning them in `modelRoles`.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `modelRoles` | record | `{}` | Map of role name → model id. |
| `modelRoleStorage` | enum | `global` | `global` saves selector role changes to the active global/profile config; `project` saves only those roles to `<cwd>/.omp/config.yml`. Missing project roles fall back to global. |
| `modelTags` | record | `{}` | Custom role/tag metadata; can introduce additional roles. |
| `modelProviderOrder` | array | `[]` | Preferred provider order when a model id is ambiguous. |
| `cycleOrder` | array | `["smol","default","slow"]` | Roles cycled by the model switcher (`Ctrl+P`). |
| `enabledModels` | array | `[]` | Allow-list of models; supports path-scoped entries. Empty = all available. |
| `enabledProviders` | array | `[]` | Foreign user-level discovery sources to load (Cursor, Codex, Claude, Gemini, OpenCode, Windsurf, GitHub); supports path-scoped entries. Project roots stay enabled regardless. |
| `disabledProviders` | array | `[]` | Disabled model providers (`anthropic`, `openai`, …) or discovery sources (`claude`, `codex`, …); supports path-scoped entries. |
| `includeModelInPrompt` | boolean | `true` | Include the active model name in the system prompt. |

### Role selector aliases

- `@<role>` selects a configured role (e.g. `@slow`).
- `*` selects the default role.
- `pi/<role>` is the legacy alias prefix (still accepted).

### Thinking & Reasoning

| Key | Type | Default | Values / notes |
| --- | --- | --- | --- |
| `defaultThinkingLevel` | enum | `high` | `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `auto`. Override per run with `--thinking`. |
| `hideThinkingBlock` | boolean | `false` | Hide thinking blocks in output (display only). `--hide-thinking` overrides. |
| `proseOnlyThinking` | boolean | `false` | Prefer prose reasoning over tool-code reasoning. |
| `thinkingBudgets.minimal` | number | `1024` | Token budget per level. |
| `thinkingBudgets.low` | number | `2048` | |
| `thinkingBudgets.medium` | number | `8192` | |
| `thinkingBudgets.high` | number | `16384` | |
| `thinkingBudgets.xhigh` | number | `32768` | |
| `thinkingBudgets.max` | number | `32768` | |
| `providers.autoThinkingMaxEffort` | enum | `xhigh` | Ceiling that `defaultThinkingLevel: auto` may resolve to (`max` lets the classifier bill the top tier). |

---

## 2. Advisor (`advisor`, `task.agentAdvisor`)

The advisor is a second model (assigned to `modelRoles.advisor`) that passively reviews each completed turn and injects notes. Enable with `advisor.enabled`, `/advisor on`, or `--advisor`.

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `advisor.enabled` | boolean | `false` | Enable the advisor runtime when `modelRoles.advisor` resolves. |
| `advisor.syncBacklog` | enum | `off` | Bounded catch-up delay: `off`, `1`, `3`, `5`. |
| `advisor.immuneTurns` | number | `3` | After a `concern`/`blocker` interrupts, route further concerns as non-interrupting asides for this many completed turns. |
| `advisor.maxNotesPerUpdate` | number | `4` | Max non-blocker advice notes accepted per advisor update (1–32). Blockers are exempt. |
| `task.agentAdvisor` | record | `{}` | Per-agent subagent advisor: agent name → `"on"` / `"off"` / advisor model pattern. |

---

## 3. Sampling (`temperature`, `tier.*`, `personality`)

A value of `-1` means "use the provider/model default" — OMP sends no parameter.

| Key | Type | Default | Values / notes |
| --- | --- | --- | --- |
| `temperature` | number | `-1` | |
| `topP` | number | `-1` | |
| `topK` | number | `-1` | |
| `minP` | number | `-1` | |
| `presencePenalty` | number | `-1` | |
| `repetitionPenalty` | number | `-1` | |
| `textVerbosity` | enum | `medium` | `low`, `medium`, `high`. |
| `tier.openai` | enum | `none` | `none`, `auto`, `default`, `flex`, `scale`, `priority`. `--service-tier` overrides per session. |
| `tier.anthropic` | enum | `none` | `none`, `priority`. |
| `tier.google` | enum | `none` | `none`, `flex`, `priority`. |
| `tier.subagent` | enum | `inherit` | `inherit`, `none`, `auto`, `default`, `flex`, `scale`, `priority`. |
| `tier.advisor` | enum | `none` | `inherit`, `none`, `auto`, `default`, `flex`, `scale`, `priority`. |
| `personality` | enum | `default` | `default`, `friendly`, `pragmatic`, `none`. |

---

## 4. Retry & Fallback (`retry`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `retry.enabled` | boolean | `true` | Retry transient provider errors (429, 5xx, timeouts). |
| `retry.maxRetries` | number | `10` | Max retries per request. |
| `retry.baseDelayMs` | number | `500` | Initial exponential backoff. |
| `retry.maxDelayMs` | number | `300000` | Backoff ceiling (5 min). `0` disables the cap. |
| `retry.modelFallback` | boolean | `true` | Fall back to another model when one is unavailable. |
| `retry.fallbackChains` | record | `{}` | Maps roles, model selectors, or `provider/*` wildcards to ordered fallback selectors. |
| `retry.fallbackRevertPolicy` | enum | `cooldown-expiry` | `cooldown-expiry` returns to the primary once its suppression window ends; `never` stays on the fallback. |
| `retry.waitForUsageReset` | boolean | `false` | Sleep until a provider-stated usage-limit reset instead of failing fast. |
| `retry.usageAwareFallback` | boolean | `false` | Treat a coding-plan model as near its limit and fall back before it hard-fails. |
| `retry.usageReservePct` | number | `10` | Remaining-percentage threshold that counts as "near its limit". |
| `retry.usageReservePolicy` | enum | `confirm` | `confirm`, `auto`, `fail-closed` — what to do when every same-provider account is inside the reserve margin. |

### Fallback chain key resolution

A key containing `/` is model-oriented and wins over roles: `provider/model-id` matches that exact model, `provider/*` matches every model of the provider. A `provider/*` **entry** keeps the failing model's id and swaps the provider. The `default` chain covers every assigned role without its own chain. Unknown models/providers or malformed chains surface as startup config warnings.

---

## 5. Tools & Approvals (`tools`, `bash`, `bashInterceptor`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `tools.approvalMode` | enum | `yolo` | `always-ask` (auto-approve read-only), `write` (auto-approve read + workspace-write), `yolo` (auto-approve all tiers). |
| `tools.approval` | record | `{}` | Per-tool policy keyed by tool name; value is `allow`, `deny`, or `prompt`. |
| `tools.format` | enum | `auto` | Tool wire format: `auto`, `native`, `glm`, `hermes`, `kimi`, `xml`, `anthropic`, `deepseek`, `harmony`, `qwen3`, `gemini`, `gemma`, `minimax`. Applies on session start. |
| `tools.maxTimeout` | number | `0` | Max tool runtime in seconds; `0` = no cap. |
| `tools.intentTracing` | boolean | `true` | Record per-call intent strings. |
| `tools.outputMaxColumns` | number | `768` | Per-line byte cap for streaming output; `0` disables. |
| `tools.artifactSpillThreshold` | number | `50` | KB of output above which the result spills to a session artifact. |
| `tools.artifactHeadBytes` | number | `20` | KB of head kept inline on spill; `0` = tail-only. |
| `tools.artifactTailBytes` | number | `20` | KB of tail kept inline on spill. |
| `tools.artifactTailLines` | number | `500` | Max tail lines kept inline on spill. |
| `bash.enabled` | boolean | `true` | Enable the bash tool. |
| `bash.allowCompoundCommands` | boolean | `false` | Evaluate flat, literal `&&` chains per segment (opt-in; conservative shell classifier). |
| `bash.direnv` | enum | `auto` | `auto` loads `.envrc`; `off` skips it. |
| `bash.direnvLoadTimeoutMs` | number | `30000` | Budget for one direnv load. |
| `bash.autoBackground.enabled` | boolean | `true` | Auto-background commands exceeding the threshold. |
| `bash.autoBackground.thresholdMs` | number | `60000` | Threshold before auto-backgrounding. |
| `bash.patterns` | array | `[]` | Ordered glob rules `{ match, approval: allow\|prompt\|deny }`; first match wins. |
| `bashInterceptor.enabled` | boolean | `false` | Redirect matching shell commands to dedicated tools. |
| `bashInterceptor.patterns` | array | `[]` | `{ pattern: regex, tool: toolName, message: string }`. |

**Bash pattern semantics.** An `allow` rule must match the whole command and cannot approve a compound line unless `bash.allowCompoundCommands: true`. `deny` and `prompt` rules may match the whole command or any tokenized segment of other compound forms (`&&`, `||`, `;`, `|`, `&`, subshells, newlines). Same-chain restrictions resolve conservatively: a `deny` wins, else a `prompt`. Patterns govern the `bash` tool only — to cover shells started through `eval`, also set `tools.approval.eval`.

**Interceptors vs patterns.** `bashInterceptor` routes commands to dedicated tools (e.g. `cat` → `read`) rather than deciding whether they may run. The named replacement tool must be available in the session or the interceptor does not block the call.

---

## 6. Shell, Eval, LSP (`eval`, `launch`, `python`, `lsp`, `computer`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `eval.py` | boolean | `true` | Enable the Python eval backend (`PI_PY=0` disables). |
| `eval.js` | boolean | `true` | Enable the JavaScript eval backend (`PI_JS=0` disables). |
| `eval.tools.enabled` | boolean | `true` | Expose `@tool` definitions to eval cells. |
| `eval.workpool.freshAgents` | boolean | `false` | Use a new agent per workpool item. |
| `eval.autoBackground.enabled` | boolean | `false` | Auto-background long eval cells. |
| `eval.autoBackground.thresholdMs` | number | `60000` | Threshold for eval auto-backgrounding. |
| `launch.enabled` | boolean | `true` | Enable the launch tool for shared long-running project processes. |
| `lsp.enabled` | boolean | `true` | Enable LSP tools, diagnostics, and formatting. |
| `lsp.lazy` | boolean | `true` | Start language servers on demand. |
| `lsp.shared` | boolean | `true` | Share language servers across sessions. |
| `lsp.formatOnWrite` | boolean | `false` | Format files after write. |
| `lsp.diagnosticsOnWrite` | boolean | `true` | Report diagnostics after write. |
| `lsp.diagnosticsOnEdit` | boolean | `false` | Report diagnostics after edit. |
| `computer.enabled` | boolean | `false` | Enable the window-aware `computer` Eval prelude. |
| `computer.display` | string | `all` | `all` composites every active display, or one numeric display ID. |
| `computer.maxWidth` | number | `3840` | Max composite screenshot width. |
| `computer.maxHeight` | number | `2400` | Max composite screenshot height. |

Per-tool toggles also exist, e.g. `glob.enabled`, `grep.enabled`, `fetch.enabled`, `browser.enabled`, `astEdit.enabled`, `astGrep.enabled`, `web_search.enabled`.

---

## 7. Edit, Read & Images (`edit`, `read`, `images`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `edit.mode` | enum | `hashline` | `replace`, `patch`, `hashline`, `apply_patch`, `sloppy`. |
| `edit.fuzzyMatch` | boolean | `true` | Fuzzy matching for anchor lines. |
| `edit.fuzzyThreshold` | number | `0.95` | Similarity threshold (0.0–1.0). |
| `edit.streamingAbort` | boolean | `false` | Abort a streaming edit on mismatch. |
| `edit.recoverInlineEdits` | boolean | `true` | Recover inline edits from malformed edit output. |
| `edit.blockAutoGenerated` | boolean | `true` | Refuse edits to lockfiles and auto-generated assets. |
| `edit.enforceSeenLines` | boolean | `false` | Require the target lines to have been read first. |
| `edit.blackbox.enabled` | boolean | `false` | Enable edit blackbox recording. |
| `edit.autoRepair.enabled` | boolean | `false` | Auto-repair common edit failures. |
| `read.defaultLimit` | number | `300` | Default line count when no selector is given. |
| `read.renderMarkdown` | boolean | `false` | Render markdown in read output. |
| `read.summarize.enabled` | boolean | `true` | Emit structural summaries on full-file code reads. |
| `read.summarize.prose` | boolean | `false` | Summarize prose/markdown files too. |
| `read.toolResultPreview` | boolean | `false` | Preview read results in the tool card. |
| `images.questionTimeoutMs` | number | `300000` | Timeout for `read <image>?q=<question>`. |
| `images.autoResize` | boolean | `true` | Auto-resize attached images. |
| `images.blockImages` | boolean | `false` | Refuse image attachments. |
| `images.describeForTextModels` | boolean | `true` | Describe images for text-only models. |
| `images.urls.enabled` | boolean | `false` | Publish images to a URL destination. |
| `images.urls.backends` | array | `["provider-files","tailscale","cloudflared","litterbox"]` | Destination backends in order. |

---

## 8. Compaction, Context & Memory (`compaction`, `memory`, `autolearn`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `compaction.enabled` | boolean | `true` | Auto-compact when context grows too large. |
| `compaction.methodOrder` | array | `[remote, snapcompact, handoff, shake, soft]` | Fallback order of compaction strategies. |
| `compaction.thresholdPercent` | number | `-1` | Context-fill percentage that triggers compaction (`-1` = reserve-based). |
| `compaction.thresholdTokens` | number | `-1` | Fixed token trigger (`-1` = percentage/reserve). |
| `compaction.keepRecentTokens` | number | `20000` | Most recent tokens always preserved untouched. |
| `compaction.midTurnEnabled` | boolean | `true` | Check thresholds at safe mid-turn tool-loop boundaries. |
| `compaction.autoContinue` | boolean | `true` | Continue after a compaction completes. |
| `compaction.idleEnabled` | boolean | `false` | Compact while idle. |
| `compaction.idleThresholdTokens` | number | `200000` | Token threshold for idle compaction. |
| `compaction.idleTimeoutSeconds` | number | `300` | Idle duration before compaction. |
| `compaction.experimentalContextManagement` | boolean | `false` | Notes-backed context windows: persistent notes + searchable raw history across windows (restart to update tools). |
| `contextPromotion.enabled` | boolean | `false` | Promote to a larger-context model on overflow instead of compacting. |
| `extendedContext` | boolean | `false` | Use advertised maximum context windows / premium long-context tiers. |
| `snapcompact.shape` | enum | `auto` | `auto` or a specific frame variant (e.g. `8on22-bw`, `11on16-bw`, `silver16-bw`, `doc-8on16-bw`). `auto` picks per model/text. |
| `memory.backend` | enum | `off` | `off`, `local`, `hindsight`, `mnemopi`, `sharpshooter`. |
| `autolearn.enabled` | boolean | `false` | Nudge the agent at stop to record lessons and mint/update managed skills. |
| `autolearn.autoContinue` | boolean | `false` | Auto-continue after an autolearn mint. |
| `autolearn.minToolCalls` | number | `5` | Minimum tool calls before autolearn triggers. |

---

## 9. UI, Terminal & Appearance (`theme`, `tui`, `display`, `statusLine`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `theme.dark` | string | `titanium` | Dark background theme. |
| `theme.light` | string | `light` | Light background theme. |
| `symbolPreset` | enum | `unicode` | `unicode`, `nerd`, `ascii`. |
| `colorBlindMode` | boolean | `false` | Shift diff additions toward blue. |
| `statusLine.preset` | enum | `default` | `default`, `minimal`, `compact`, `full`, `nerd`, `ascii`, `custom`. |
| `statusLine.separator` | enum | `powerline-thin` | `powerline`, `powerline-thin`, `slash`, `pipe`, `block`, `none`, `ascii`. |
| `statusLine.contextLine` | enum | `embedded` | Context-line placement mode. |
| `statusLine.sessionAccent` | boolean | `true` | Per-session accent color. |
| `statusLine.transparent` | boolean | `false` | Transparent status line background. |
| `statusLine.compactThinkingLevel` | boolean | `true` | Compact thinking-level label. |
| `statusLine.showHookStatus` | boolean | `true` | Show hook status. |
| `statusLine.leftSegments` | array | `[]` | Explicit left segment list. |
| `statusLine.rightSegments` | array | `[]` | Explicit right segment list. |
| `composer.shape` | string | `band` | Composer prompt shape. |
| `terminal.showImages` | boolean | `true` | Render inline images. |
| `terminal.showProgress` | boolean | `false` | Show progress indicators. |
| `tui.hyperlinks` | enum | `auto` | `off`, `auto`, `always`. |
| `tui.textSizing` | boolean | `false` | Dynamic terminal text sizing. |
| `tui.renderMermaid` | boolean | `true` | Render Mermaid diagrams. |
| `tui.reactions` | boolean | `true` | Animated status reactions. |
| `tui.titleState` | boolean | `true` | Reflect state in the terminal title. |
| `tui.tight` | boolean | `false` | Tighter vertical spacing. |
| `tui.resizeScrollback` | enum | `rebuild` | `append`, `rebuild`, `preserve`. |
| `tui.maxInlineImageColumns` | number | `100` | Inline image column cap. |
| `tui.maxInlineImageRows` | number | `20` | Inline image row cap. |
| `tui.maxInlineImages` | number | `8` | Max inline images per message. |
| `tui.vimMode` | boolean | `false` | Modal prompt editing (Escape → Normal; hjkl, operators, text objects, Visual). |
| `tui.vimModeDisplay` | enum | `text` | `text`, `icon`, `none` — how the mode shows in the status line. |
| `display.shimmer` | enum | `classic` | `classic`, `kitt`, `disabled`. |
| `display.smoothStreaming` | boolean | `true` | Smooth token streaming. |
| `display.hideToolActivity` | boolean | `false` | Hide in-flight tool activity. |
| `display.showTokenUsage` | boolean | `false` | Show token usage. |
| `display.showTurnTime` | boolean | `false` | Show per-turn duration. |
| `display.cacheMissMarker` | boolean | `false` | Mark cache misses. |
| `display.collapseCompacted` | boolean | `true` | Collapse compacted history. |

---

## 10. Interaction, Startup & Delegation (`startup`, `plan`, `todo`, `task`, `loop`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `startup.quiet` | boolean | `false` | Suppress startup chrome. |
| `startup.showSplash` | boolean | `false` | Show the first-run splash animation. |
| `startup.setupWizard` | boolean | `true` | Run the setup wizard on first launch. |
| `startup.checkUpdate` | boolean | `true` | Check for updates at startup. |
| `startup.changelogMode` | enum | `summary` | `summary`, `expanded`, `hidden`. |
| `steeringMode` | enum | `one-at-a-time` | Queued steering delivery: `all`, `one-at-a-time`. |
| `followUpMode` | enum | `one-at-a-time` | Follow-up message delivery: `all`, `one-at-a-time`. |
| `interruptMode` | enum | `immediate` | `immediate`, `wait`. |
| `doubleEscapeAction` | enum | `rewind` | `rewind`, `tree`, `none`. |
| `autoResume` | boolean | `false` | Auto-resume the most recent session in the cwd. |
| `composer.recallClearedDrafts` | boolean | `true` | Recall drafts cleared with `Ctrl+C` via `Up`. |
| `plan.enabled` | boolean | `true` | Enable plan mode (`/plan`). |
| `plan.defaultOnStartup` | boolean | `false` | Start fresh interactive sessions in plan mode. |
| `plan.autosave` | boolean | `false` | Autosave approved plans when plan mode completes. |
| `plan.autosaveDir` | string | _(unset)_ | Autosave directory; empty uses `<project>/.omp/plans/`. |
| `todo.enabled` | boolean | `true` | Enable structured todo tracking. |
| `loop.mode` | enum | `prompt` | Between `/loop` iterations: `prompt` (re-submit), `compact`, `reset` (new session). |
| `loop.conditionTimeoutMs` | number | `30000` | Max wait for a `/loop --while`/`--until` condition command (`0` = unlimited). |
| `ask.timeout` | number | `0` | Seconds before an `ask` prompt times out; `0` = none. |
| `ask.notify` | enum | `on` | `on`, `off`. |
| `task.maxConcurrency` | number | `32` | Max parallel subagents in a batch. |
| `task.batch` | boolean | `true` | Allow batched subagent dispatch. |
| `task.eager` | enum | `default` | `default`, `preferred`, `always`. |
| `task.enableEffort` | boolean | `false` | Allow per-task effort hints. |
| `task.enableLsp` | boolean | `false` | Give subagents LSP tools. |
| `task.maxRecursionDepth` | number | `2` | Subagent recursion depth cap. |
| `task.maxRuntimeMs` | number | `0` | Subagent runtime cap (`0` = none). |
| `task.softRequestBudget` | number | `200` | Soft request budget per subagent. |
| `task.maxEffort` | enum | `max` | Ceiling for subagent thinking effort. |
| `task.isolation.enabled` | boolean | `false` | Run subagents in an isolated filesystem. |
| `task.isolation.apply` | boolean | `true` | Apply isolated changes back. |
| `task.isolation.merge` | enum | `patch` | `patch`, `branch`. |
| `task.isolation.commits` | enum | `generic` | `generic`, `ai`. |
| `task.disabledAgents` | array | `[]` | Agent names to hide. |
| `task.agentModelOverrides` | record | — | Per-agent model assignment. |
| `task.agentPrewalk` | record | `{}` | Per-agent prewalk toggle/model. |
| `task.prewalk` | boolean | `false` | Arm prewalk for the bundled generic `task` subagent. |

---

## 11. Providers & Services (`providers`, `exa`, `searxng`, `auth`, `mcp`, `secrets`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `providers.webSearchOrder` | array | `[]` | Priority order of `web_search` providers (`perplexity`, `gemini`, `anthropic`, `codex`, `xai`, `zai`, `exa`, `tinyfish`, `jina`, `kagi`, `tavily`, `firecrawl`, `brave`, `kimi`, `parallel`, `synthetic`, `searxng`, `startpage`, `duckduckgo`, `ecosia`, `google`, `mojeek`, `public`). Empty = built-in order. |
| `providers.webSearchExclude` | array | `[]` | Providers `web_search` must never use. |
| `providers.webSearchTimeoutSeconds` | number | `60` | Per-provider transport timeout before the chain advances. |
| `providers.webSearchGeminiModel` | string | _(unset)_ | Gemini model for Google Search grounding (default `gemini-2.5-flash`). |
| `providers.imageOrder` | array | `[]` | Image-generation provider priority. |
| `providers.fetch` | enum | `auto` | `auto`, `native`, `trafilatura`, `lynx`, `parallel`, `firecrawl`, `jina`. |
| `providers.tinyModel` | enum | `online` | `online` or a local tiny model (`lfm2.5-230m`, `lfm2.5-350m`, `falcon-h1-90m`). |
| `providers.tinyModelDevice` | enum | `default` | ONNX provider or `mlx`; `PI_TINY_DEVICE` overrides. |
| `providers.tinyModelDtype` | enum | `default` | ONNX precision; `PI_TINY_DTYPE` overrides. |
| `providers.openaiWebsockets` | enum | `auto` | `auto`, `off`, `on`. |
| `providers.openrouterVariant` | enum | `default` | `default`, `nitro`, `floor`, `online`, `exacto`. |
| `providers.kimiApiFormat` | enum | `auto` | `auto`, `openai`, `anthropic`. |
| `providers.cacheRetention` | enum | `auto` | `auto`, `short`, `long`, `none`. |
| `providers.maxInFlightRequests` | record | `{}` | Per-provider concurrency limits. |
| `provider.appendOnlyContext` | enum | `auto` | `auto`, `on`, `off`. |
| `exa.enabled` | boolean | `true` | Enable the Exa web-search provider. |
| `exa.searchDelayMs` | number | `1000` | Minimum delay between Exa requests. |
| `searxng.endpoint` | string | _(unset)_ | SearXNG instance URL. |
| `searxng.token` | string | _(unset)_ | SearXNG token. |
| `auth.broker.url` | string | _(unset)_ | Auth-broker URL; `OMP_AUTH_BROKER_URL` overrides. |
| `auth.broker.token` | string | _(unset)_ | Auth-broker token; `OMP_AUTH_BROKER_TOKEN` overrides. |
| `mcp.enableProjectConfig` | boolean | `true` | Load project-scoped MCP config. |
| `mcp.renderMarkdownResults` | boolean | `true` | Render markdown MCP results. |
| `mcp.notifications` | boolean | `false` | Surface MCP server notifications. |
| `mcp.notificationDebounceMs` | number | `500` | Notification debounce. |
| `secrets.enabled` | boolean | `false` | Obfuscate configured secrets and credential-shaped tokens before provider requests. |
| `commit.mapReduceEnabled` | boolean | `true` | Map-reduce large diffs for changelogs. |
| `commit.mapReduceThreshold` | number | `5000` | Diff size before map-reduce. |
| `commit.cacheEnabled` | boolean | `true` | Cache commit generations. |
| `commit.cacheTtlDays` | number | `14` | Commit cache TTL. |
| `extensionHandlers.toolCallTimeoutMs` | number | `30000` | Active-work timeout for extension `tool_call` handlers. |

---

## 12. Environment Overrides

Each variable is read by the feature that owns the value and is never written back to `config.yml`.

| Env var | Overrides setting | Notes |
| --- | --- | --- |
| `PI_SMOL_MODEL` | `modelRoles.smol` | Also `--smol`. |
| `PI_SLOW_MODEL` | `modelRoles.slow` | Also `--slow`. |
| `PI_PLAN_MODEL` | `modelRoles.plan` | Also `--plan`. |
| `PI_NO_PTY=1` | (disables PTY bash) | Equivalent to `--no-pty`. |
| `PI_PY` | `eval.py` | `PI_PY=0` disables. |
| `PI_JS` | `eval.js` | `PI_JS=0` disables. |
| `PI_TINY_DEVICE` | `providers.tinyModelDevice` | |
| `PI_TINY_DTYPE` | `providers.tinyModelDtype` | |
| `OMP_AUTH_BROKER_URL` | `auth.broker.url` | |
| `OMP_AUTH_BROKER_TOKEN` | `auth.broker.token` | |
| `PI_CODING_AGENT_DIR` | (relocates agent dir) | Moves `config.yml`, `agent.db`, and the whole agent base. |
| `PI_CONFIG_FILES` | CLI config overlays | Platform path-list (`:` Unix, `;` Windows); loads before `--config` overlays. |
| `OMP_MCP_TIMEOUT_MS` | all MCP server timeouts | `0` disables client-side MCP timeouts. |
| `PI_PROFILE` / `OMP_PROFILE` | active profile | Selects `~/.omp/profiles/<name>/agent/`. |
| `PI_EDIT_VARIANT` | `edit.mode` | Overrides the edit mode for the process. |
| `PI_HARDWARE_CURSOR` | (cursor shape) | Use the real terminal cursor via DECSCUSR in Vim mode. |

---

## 13. Path-Scoped Arrays

`enabledModels`, `enabledProviders`, and `disabledProviders` accept path-scoped entries alongside bare strings, so one global config behaves differently per directory:

```yaml
enabledModels:
  - claude-sonnet-4-5
  - path: ~/work/high-context
    models:
      - anthropic/claude-opus-4-5

disabledProviders:
  - ollama
  - paths:
      - ~/projects/sensitive
    providers:
      - anthropic
```

Accepted path keys: `path`, `paths`, `pathPrefix`, `pathPrefixes`. Accepted value keys: `models` (for `enabledModels`), `providers` (for `enabledProviders`/`disabledProviders`), or `values`/`items` for any. A scoped entry applies when the cwd **is** the path or is **under** it. Path scoping resolves after the layer merge.

---

## 14. Legacy Migrations

Applied automatically at load time:

| Old | New |
| --- | --- |
| `queueMode` | `steeringMode` |
| flat `theme: "<name>"` | `theme.dark` / `theme.light` |
| `inspect_image.timeoutMs` | `images.questionTimeoutMs` |
| `inspect_image.enabled` / `.mode` | removed |
| `task.isolation.mode: none` | `task.isolation.enabled: false` |
| `task.isolation.mode: <backend>` | `task.isolation.enabled: true` + `isolation.backend` |
| legacy isolation backends (`worktree`, `fuse-overlay`, `fuse-projfs`) | `rcopy`, `overlayfs`, `projfs` |
| `ask.timeout` in ms (> 1000) | seconds |
| `task.simple` | removed |
| `lastChangelogVersion` | moved to a marker file |

Startup also migrates `~/.omp/agent/settings.json` (renamed `.bak`) and settings persisted in `agent.db` into `config.yml` once, only when neither `config.yml` nor `config.yaml` exists.
