---
name: opencodereview-cli
description: Use the OpenCodeReview CLI (opencodereview / ocr) for AI code review of workspace changes, branches, and commits, plus full-file scans, session resume/inspection, rule debugging, delegation mode, configuration, and troubleshooting. TRIGGERS - any use or mention of opencodereview, open-code-review, open code review, ocr review, ocr scan, ocr delegate, ocr session, ocr config, ocr rules, or Alibaba OpenCodeReview. Load before invoking opencodereview/ocr, including when another workflow calls it.
---

# OpenCodeReview CLI

OpenCodeReview reviews Git diffs and scans source using the configured LLM provider. Select the review scope explicitly; never stage or commit merely to make a review command work.

The binary in this environment is `~/.local/share/mise/shims/opencodereview`. Upstream docs call it `ocr` (`ocr` and `opencodereview` are the same command; `review`/`r`, `scan`/`s`, `session`/`sessions`, `delegate`/`d`, `viewer`/`v` are aliases). Examples below use `ocr` to match upstream; substitute the full shim path when `ocr` is not on PATH:

```bash
OCR=~/.local/share/mise/shims/opencodereview
$OCR --version
$OCR review --help
```

## Preconditions and safety

1. Check `$OCR --version`. This skill was checked against **open-code-review v1.12.6 (7a571b7) linux/amd64, built 2026-09-18**. Use the installed subcommand help when versions differ or a flag is uncertain (`review`, `scan`, `session`, `config`, `rules`, `delegate`, `llm`, `viewer` each have `--help`).
2. Before Git-based reviews, inspect from the repository root:
   ```bash
   git rev-parse --show-toplevel
   git status --short
   git diff --stat HEAD
   git diff --cached --stat
   ```
   In short status, the first column is the index, the second is the working tree; `??` means untracked.
3. Requires **Git >= 2.41**. OCR shells out to Git for diffs, search, and merge-base computation.
4. Respect existing provider/model configuration. Reviews send diffs **and retrieved file context** to the configured endpoint and incur cost. A local CLI does not mean local LLM inference. Do not send secrets or restricted code to an unapproved provider.
5. Config lives at `~/.opencodereview/config.json` and may contain API keys. Sessions live under `~/.opencodereview/sessions/`. Do not dump the config file, print API keys, or paste session internals with secrets into tool output. For connectivity checks prefer `ocr llm test` over echoing config values.
6. A mention of OpenCodeReview triggers this guidance, **not permission to execute every command**. A review request does not authorize staging, committing, pushing, amending, changing configuration, installing anything, launching the viewer server, uploading results, or persisting findings to external memory.
7. Review/scan commands persist session logs under `~/.opencodereview/sessions/`. Inspect `git status --short` afterward and do not stage generated artifacts automatically.
8. Do not rerun the same review or scan when the reviewed file contents have not changed. A review followed by type checking, staging, or other non-file mutations does not require another run. Reviews are not free. Run a new command only for changed input files, an intentionally different review scope, or an explicit user request. Prefer `--resume <session-id>` for interrupted range/commit reviews instead of starting over.
9. Keep output outside the reviewed tree (e.g. `/tmp/ocr-review.json`) when using `--output`.

## Choose the right scope

Mode flags are **mutually exclusive**: pass either `--from`/`--to`, or `--commit`, or neither (workspace mode). Mixing them is a hard error.

| Situation | Command | Actual review input / caveat |
|---|---|---|
| Workspace changes, before commit | `ocr review` | Staged + unstaged via `git diff HEAD`, plus untracked files (via `git ls-files --others --exclude-standard`, read as full-file additions). This is the pre-commit default. Unlike some tools, there is no separate `--staged`/`--unstaged` mode. |
| Branch / PR review | `ocr review --from main --to feature-branch` | `merge-base(main, feature-branch)..feature-branch`: only what the feature branch introduced. Replace with the actual target branch. |
| A single commit | `ocr review --commit abc123` (`-c`) | `git show abc123`: what that commit introduced, versus its parent. No checkout needed. |
| Resume interrupted range review | `ocr review --from main --to feature --resume <session-id>` | Requires same `--from`/`--to` as the original session. List candidates with `ocr session list`. |
| Resume interrupted commit review | `ocr review --commit abc123 --resume <session-id>` | Requires same `--commit` as the original session. |
| Preview without LLM cost | `ocr review --preview` (or `-p`) | Runs file selection/filtering only; prints file list and exclusion reasons. Combine with scope flags, e.g. `ocr review -c abc123 -p`. Cannot combine `--preview` with `--resume`. |
| Whole project / no useful diff | `ocr scan` | Full-file scan of the whole repo, not a patch review. |
| Directory or files, no diff needed | `ocr scan --path internal/agent` or `--path a.go,b.go` | Comma-separated repo-relative dirs/files. |
| Preview a scan without LLM cost | `ocr scan --preview` | File selection only. |
| Resume interrupted scan | `ocr scan --resume <session-id>` | Full-file scans support resume; workspace `review` does not. |

### Workspace mode notes

- Bare `ocr review` already covers staged, unstaged, **and** untracked changes. Do not stage files or use `git add -N` merely to expand coverage.
- To narrow scope, prefer `--exclude` (comma-separated gitignore-style patterns, merged with rule.json excludes) or a range/commit scope — not index manipulation:
  ```bash
  ocr review --exclude '**/generated/*,**/testdata/*'
  ```
- Untracked files are treated as full-file additions. Confirm they appear in `--preview` output; path/extension rules can still filter them.

### Range / commit notes

- Verify refs exist before reviewing:
  ```bash
  git rev-parse --verify main
  git rev-parse --verify feature-branch
  git merge-base main feature-branch
  git log --oneline main..feature-branch
  ```
- Shallow CI clones may lack history for merge-base computation; fetch/unshallow when allowed and needed.
- Fixing a historical finding does not authorize amend, reset, rebase, or force-push. Review new working changes separately with workspace mode.

## Run, interpret, and verify

For agent-readable results (suppress progress lines, emit only the summary/JSON):

```bash
ocr review --format json --audience agent
ocr review --from main --to feature-branch --format json --audience agent
ocr review --commit abc123 --format json --audience agent
```

For a local machine-readable artifact outside the reviewed tree:

```bash
ocr review --from main --to feature-branch --format json --audience agent --output /tmp/ocr-review.json
```

- Supported review/scan formats: `text`, `json`, `sarif`. Delegate mode supports only `text`/`json` (no SARIF).
- `--audience human` (default) streams progress lines to stderr for json/sarif; `--audience agent` quiets stdout to the final summary/JSON. Use `agent` in CI or when piping to another agent.
- MUST NOT pipe `ocr` output through `head`, `tail`, `grep`, `sed`, `awk`, `cut`, `less`, `more`, or any similar filtering/paging tool. Run the command bare so all output is captured in full. To narrow results, use OCR's own flags instead (e.g. `--format json`, `--audience agent`, `--output <file>`, `--severity`/`--category`, `--limit`) — never shell-level filtering.
- JSON success envelope includes `status`, `summary` (`files_reviewed`, `comments`, `total_tokens`, `input_tokens`, `output_tokens`, `elapsed`), `comments[]` (`path`, `content`, `start_line`, `end_line`, `existing_code`, `suggestion_code`, `thinking`), optional `warnings`, `session_id`, and `resume` metadata on resumed runs.
- When no files are eligible, JSON emits a `skipped` envelope (`"status": "skipped"`, `"comments": []`, e.g. `"No supported files changed."`). That means no review occurred — not a clean bill of health. Use `--preview` to confirm scope.
- An empty `comments` array with `status: success` means the review completed with no findings. Do not treat it as an error or rerun just because the summary text is terse.
- Exit **0 means completed** (possibly zero comments, possibly non-fatal per-file warnings listed inline or in `warnings`). Exit **1 means fatal** — bad flags, unresolvable LLM endpoint, all sub-agents failed, etc. Preserve the actual status and stderr diagnostics. There is no findings-based exit-2 gate as in some other reviewers.
- Useful tuning knobs (prefer narrowing scope first): `--effort low|medium|high`, `--concurrency` (default 8), `--timeout` minutes per task (default 15), `--max-tools`, `--max-tokens`, `--max-tokens-budget 0` = unlimited, `--no-filter` to keep comments without LLM post-filtering, `--background` / `--background-file` for requirement context, `--rule` for a custom JSON rule file, `--model` / `--provider` per-run overrides.
- Verify actionable findings against source and tests; LLM findings are not automatically correct. Do not blindly apply suggestions.
- After authorized fixes, run project tests/lint/typechecks and rerun the correct scope. A committed diff does not include an uncommitted fix.
- Report: command and scope/refs, coverage gaps (filtered/skipped files, warnings), exit status, actionable findings with file/line, session ID when relevant, and checks actually run.

## Sessions: list, inspect, compare, resume

```bash
ocr session list
ocr session list --limit 50 --json
ocr session show <session-id>
ocr session show --json <session-id>
ocr session comments <session-id>
ocr session comments --json --severity critical,high <session-id>
ocr session compare <before-session-id> <after-session-id>
ocr session compare --json <before-session-id> <after-session-id>
```

- `list` shows recent sessions for the current repo (`--repo <path>` to target another repo, `--limit 0` for unlimited).
- `show` prints session metadata and per-file checkpoint status — use it to confirm a session matches the intended `--from`/`--to` or `--commit` before `--resume`.
- `comments` reprints persisted comments (filter with `--severity critical,high,medium,low` and/or `--category`, e.g. `--category bug,security`).
- `compare` groups findings into new/persisting/resolved/not-reviewed, matched on path, category, and snippet (line moves still count as persisting). Use it to verify a fix round.
- Resume rules: workspace reviews cannot be resumed; range must match `--from`/`--to`; commit must match `--commit`; `--preview` + `--resume` is rejected.

## CI and SARIF

```bash
ocr review --from main --to feature-branch --audience agent --format sarif --output /tmp/ocr-results.sarif
```

- Capture the exit status; exit 0 with warnings still means completed-with-warnings — inspect the output, do not mask failure with `|| true`.
- Generating SARIF is local. Publishing it (GitHub Code Scanning upload, PR comments, Gerrit votes) requires separate tooling/authorization — a review request alone does not authorize uploads.
- Use trusted CI configuration and least-privilege credentials. OCR can load project/global rule files and custom tool configs; do not assume a review-only job makes untrusted configuration safe.
- For deterministic agent pipelines, see Delegation mode below.

## Configuration, authentication, and connectivity

```bash
ocr config provider          # interactive provider setup (select built-in/custom, enter key, pick model, tests connectivity)
ocr config model             # interactive model switch
ocr config set provider anthropic
ocr config set model claude-opus-4-6
ocr config set providers.anthropic.api_key "$ANTHROPIC_API_KEY"
ocr llm providers            # list built-in providers (name, protocol, base URL)
ocr llm test                 # verify the resolved endpoint with a canned chat request
```

- Non-interactive/CI setup uses `ocr config set <key> <value>`. Built-in providers need only `provider`, `model`, and the provider API key (or its env var, e.g. `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `DEEPSEEK_API_KEY`). Custom providers require at least `custom_providers.<name>.url` and `custom_providers.<name>.protocol` (`anthropic`, `openai`, or `openai-responses`), plus `model` and a non-empty `api_key`.
- `ocr config unset custom_providers.<name>` deletes a custom provider (clears active `provider`/`model` if it was active). Only that key pattern is supported.
- `language` controls comment language: `ocr config set language English` (or `中文`, etc.).
- Per-run overrides without touching config: `ocr review --provider anthropic --model claude-opus-4-6`.
- If the endpoint is already provided via environment (e.g. existing `ANTHROPIC_*` or `OCR_LLM_*` variables, CC-Switch proxy URLs), no config file changes are needed — verify with `ocr llm test`.
- Timeout knobs: per-provider `timeout_sec` in config JSON (manual edit only, not via `config set`), legacy `llm.timeout_sec`, or `OCR_LLM_TIMEOUT` env override. Slow local models (e.g. Ollama) may need more than the 300s default.
- Only when setup is requested: run interactive `config provider`/`config model` (requires a TTY — never in CI), or `config set` for scripts. Have the user enter secrets themselves; never commit keys or literal tokens in commands/YAML. Use secret-managed env vars.
- `extra_body` (`ocr config set providers.anthropic.extra_body '{"thinking":{"type":"disabled"}}'`) sends vendor-specific request fields without patching source.

## Additional workflows (use only when relevant)

```bash
ocr scan --path internal/agent --format json --audience agent
ocr scan --path internal/agent/agent.go,internal/diff/scan.go --exclude '**/generated/*'
ocr scan --no-plan --no-dedup --no-summary   # skip PLAN pre-pass, DEDUP task, PROJECT_SUMMARY task
ocr rules check src/main/java/com/example/Foo.java
ocr rules check --rule custom.json src/main/resources/mapper/UserMapper.xml
ocr delegate preview --from main --to feature-branch
ocr delegate preview --format json
ocr delegate rule src/main.go src/handler.go
ocr viewer                                   # local WebUI for past sessions (default localhost:5483)
ocr viewer --addr :3000 --open=never
ocr version
```

- `scan` flags mirror `review` plus `--path` (whole repo by default), `--batch none|by-language|by-directory`, `--no-plan`, `--no-dedup`, `--no-summary`.
- `rules check` walks the four-layer chain (custom → project → global → system) and prints source layer, matched glob, and rule text. Use it to debug "why isn't my custom rule firing?" or to confirm coverage before a review.
- **Delegation mode** (no OCR LLM config required): OCR handles file selection and rule resolution; the host coding agent performs the review with its own model. `delegate preview` outputs the reviewable file list with mode/ref metadata; `delegate rule <paths...>` outputs resolved rules grouped by content. Accepts the same scope flags (`--from`/`--to`, `--commit`, `--exclude`, `--rule`, `--background`/`--background-file`).
- `viewer` starts a long-running HTTP server for browsing `~/.opencodereview/sessions/`. Do not launch it for a normal CLI review; never bind beyond localhost without explicit authorization.
- `session comments`/`compare` support `--json` for machine-readable post-review analysis without rerunning the LLM.

## Sources and version caveats

Checked against installed `open-code-review v1.12.6` help and upstream docs at time of writing:

- [Repository / README](https://github.com/alibaba/open-code-review)
- [CLI reference](https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/cli-reference.md)
- [Configuration](https://github.com/alibaba/open-code-review/blob/main/pages/src/content/docs/en/configuration.md)
- [Docs site](https://open-codereview.ai/docs): quickstart, installation, CLI reference, review rules, configuration, delegation mode, CI/CD integration, session viewer, MCP server, coding-agent integrations

Upstream CLI examples sometimes use `ocr` while distribution packages may install a different binary name (here: `~/.local/share/mise/shims/opencodereview`). Prefer installed `--help` for supported syntax; the local binary name does not change flag semantics. Version 1.12.x help reports additional flags beyond the published CLI-reference snapshot (`--effort`, `--exclude`, `--background`/`--background-file`, `--no-filter`, token-budget knobs, `session comments`/`compare`, `delegate`, `scan` batch/plan/dedup/summary controls) — trust installed help when they disagree.
