# OMP Settings Catalog & Reference

This reference documents the complete settings schema, types, default values, and behaviors for Oh My Pi (`config.yml`).

## Quick CLI Reference

```bash
omp config list                        # List all effective settings with types and descriptions
omp config list --json                 # Export settings as JSON
omp config get <key>                   # Get single key value (e.g. omp config get compaction.thresholdPercent)
omp config set <key> <value>           # Set single key value
omp config reset <key>                 # Reset single key to its schema default
omp config path                        # Output path to active agent directory
```

---

## 1. Models & Roles (`modelRoles`, `modelTags`, `cycleOrder`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `modelRoles` | record | `{}` | Map of role name -> model identifier. Built-in roles: `default`, `smol`, `slow`, `plan`, `task`, `vision`, `designer`, `commit`, `tiny`, `advisor`. Accepts thinking suffix `:minimal`, `:low`, `:medium`, `:high`, `:xhigh`, `:max`. |
| `modelRoleStorage` | enum | `global` | `global` saves selector changes to global `config.yml`; `project` saves to `<cwd>/.omp/config.yml`. |
| `modelTags` | record | `{}` | Custom metadata tags and aliases for models. |
| `modelProviderOrder` | array | `[]` | Preferred provider order when model ID is ambiguous across providers. |
| `cycleOrder` | array | `["smol","default","slow"]` | Roles cycled by `Ctrl+P` (`app.model.cycleForward`). |
| `enabledModels` | array | `[]` | Allowlist of models or path-scoped objects. Empty means all models enabled. |
| `disabledProviders` | array | `[]` | Disabled model providers (`anthropic`, `openai`, etc.) or discovery sources (`claude`, `codex`, `opencode`). |
| `includeModelInPrompt` | boolean | `true` | Include active model name in system prompt. |

### Thinking & Reasoning Configuration

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `defaultThinkingLevel` | enum | `high` | Default reasoning budget level: `minimal`, `low`, `medium`, `high`, `xhigh`, `max`, `auto`. |
| `hideThinkingBlock` | boolean | `false` | Hide reasoning blocks in terminal output (display only). |
| `proseOnlyThinking` | boolean | `false` | Instruct model to use thinking for prose reasoning rather than tool code snippets. |
| `thinkingBudgets.minimal` | number | `1024` | Token budget for `minimal`. |
| `thinkingBudgets.low` | number | `2048` | Token budget for `low`. |
| `thinkingBudgets.medium` | number | `8192` | Token budget for `medium`. |
| `thinkingBudgets.high` | number | `16384` | Token budget for `high`. |
| `thinkingBudgets.xhigh` | number | `32768` | Token budget for `xhigh`. |
| `thinkingBudgets.max` | number | `32768` | Token budget for `max`. |

---

## 2. Retry & Fallback (`retry`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `retry.enabled` | boolean | `true` | Enable automatic retry of transient errors (429, 5xx, timeouts). |
| `retry.maxRetries` | number | `10` | Maximum retry attempts per request. |
| `retry.baseDelayMs` | number | `500` | Initial exponential backoff delay in ms. |
| `retry.maxDelayMs` | number | `300000` | Maximum backoff delay cap in ms (5 minutes). |
| `retry.modelFallback` | boolean | `true` | Fall back to alternative models when active model is unavailable. |
| `retry.fallbackChains` | record | `{}` | Map of role names, `provider/model-id`, or `provider/*` wildcards to ordered fallback model lists. |
| `retry.fallbackRevertPolicy` | enum | `cooldown-expiry` | `cooldown-expiry` reverts to primary after suppression window; `never` stays on fallback. |

---

## 3. Tools & Approvals (`tools`, `bash`, `edit`, `read`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `tools.approvalMode` | enum | `yolo` | `yolo` (auto-approve all), `write` (auto-approve read + workspace write), `always-ask` (ask before any modification). |
| `tools.approval` | record | `{}` | Per-tool policy map (`toolName: allow\|prompt\|deny`). |
| `tools.format` | enum | `auto` | Wire format for tool calls: `auto`, `native`, `xml`, `anthropic`, `harmony`, `kimi`, `glm`, `qwen3`, `minimax`. |
| `tools.maxTimeout` | number | `0` | Max execution time in seconds for tools (`0` = no timeout). |
| `tools.intentTracing` | boolean | `true` | Record intent strings for tool invocations. |
| `tools.outputMaxColumns` | number | `768` | Byte cap per line for streaming output. |
| `tools.artifactSpillThreshold` | number | `50` | Output size in KB above which result spills to session artifact. |
| `bash.enabled` | boolean | `true` | Enable the bash execution tool. |
| `bash.autoBackground.enabled` | boolean | `true` | Auto-background commands exceeding execution threshold. |
| `bash.autoBackground.thresholdMs` | number | `60000` | Threshold in ms before auto-backgrounding. |
| `bash.patterns` | array | `[]` | Ordered glob pattern rules with `match` and `approval: allow\|prompt\|deny`. |
| `bashInterceptor.enabled` | boolean | `false` | Intercept shell commands matching regex and redirect to dedicated tools. |
| `bashInterceptor.patterns` | array | `[]` | Array of `{ pattern: regex, tool: toolName, message: string }`. |
| `edit.mode` | enum | `hashline` | Edit tool mode: `hashline`, `apply_patch`, `patch`, `replace`. |
| `edit.fuzzyMatch` | boolean | `true` | Enable fuzzy matching for anchor lines. |
| `edit.fuzzyThreshold` | number | `0.95` | Similarity threshold for fuzzy edits (0.0 - 1.0). |
| `edit.blockAutoGenerated` | boolean | `true` | Refuse edits to lockfiles and auto-generated assets. |
| `read.defaultLimit` | number | `300` | Default line count when no selector is specified. |
| `read.summarize.enabled` | boolean | `true` | Emit structural summary on full-file code reads. |
| `read.summarize.prose` | boolean | `false` | Enable summaries for prose/markdown files. |

---

## 4. Compaction, Context & Memory (`compaction`, `memory`, `autolearn`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `compaction.enabled` | boolean | `true` | Enable automatic context compaction. |
| `compaction.methodOrder` | array | `[remote, snapcompact, handoff, shake, soft]` | Fallback order of compaction strategies. |
| `compaction.thresholdPercent` | number | `-1` | Context window fill percentage that triggers compaction (`-1` = reserve-based). |
| `compaction.thresholdTokens` | number | `-1` | Fixed token count trigger (`-1` = use percentage/reserve). |
| `compaction.keepRecentTokens` | number | `20000` | Number of most recent tokens always preserved untouched. |
| `compaction.midTurnEnabled` | boolean | `true` | Check and trigger compaction between multi-step tool loops. |
| `snapcompact.shape` | enum | `auto` | Visual compaction image formatting: `auto`, `grid`, `strip`. |
| `memory.backend` | enum | `off` | Long-term memory backend: `off`, `local`, `hindsight`, `mnemopi`. |
| `autolearn.enabled` | boolean | `false` | Nudge agent at stop to record lessons and create/update managed skills. |
| `autolearn.minToolCalls` | number | `5` | Minimum tool calls in turn before autolearn triggers. |

---

## 5. UI, Terminal & Appearance (`theme`, `tui`, `terminal`, `statusLine`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `theme.dark` | string | `titanium` | Dark background theme (e.g. `titanium`, `tokyo-night`, `dracula`, `nord`). |
| `theme.light` | string | `light` | Light background theme (e.g. `light`, `solarized-light`). |
| `symbolPreset` | enum | `unicode` | Icon set: `unicode`, `nerd`, `ascii`. |
| `colorBlindMode` | boolean | `false` | Use high-contrast blue for diff additions. |
| `tui.hyperlinks` | enum | `auto` | File and URL terminal hyperlinks: `auto`, `on`, `off`. |
| `tui.textSizing` | boolean | `false` | Dynamic terminal text sizing adjustment. |
| `statusLine.preset` | enum | `default` | Status line layout: `default`, `minimal`, `compact`, `full`, `nerd`, `ascii`, `custom`. |
| `statusLine.separator` | enum | `powerline-thin` | Segment separator: `powerline`, `powerline-thin`, `slash`, `pipe`, `block`, `none`. |
| `display.shimmer` | enum | `classic` | Model thinking shimmer animation: `classic`, `subtle`, `off`. |

---

## 6. Interaction & Task Supervision (`plan`, `todo`, `advisor`, `task`)

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `plan.enabled` | boolean | `true` | Enable plan mode workflow (`/plan`). |
| `plan.defaultOnStartup` | boolean | `false` | Start new interactive sessions in plan mode. |
| `todo.enabled` | boolean | `true` | Enable structured todo list tracking (`todo` tool and UI overlay). |
| `steeringMode` | enum | `one-at-a-time` | Queued steering message delivery: `all` or `one-at-a-time`. |
| `interruptMode` | enum | `immediate` | Interruption handling: `immediate` or `wait`. |
| `doubleEscapeAction` | enum | `tree` | Action on double-ESC: `tree` (session tree), `branch`, `none`. |
| `advisor.enabled` | boolean | `false` | Enable background advisor reviewer model. |
| `advisor.syncBacklog` | enum | `off` | Advisor backlog synchronization: `off`, `1`, `3`, `5`. |
| `task.maxConcurrency` | number | `4` | Maximum parallel subagents allowed in batch execution. |
| `task.isolation.mode` | enum | `none` | Subagent filesystem isolation: `none`, `rcopy`, `overlayfs`, `projfs`. |
