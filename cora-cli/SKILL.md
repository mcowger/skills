---
name: cora-cli
description: Use the Cora CLI (cora / cora-code) for AI code review before staging, after staging, after committing, before pushing, and for branch or PR reviews, scans, code intelligence, configuration, and troubleshooting. TRIGGERS - any use or mention of the cora CLI, cora review, cora scan, cora commit, cora index, cora config, cora hooks, cora-code, or CodeCora CLI. Load before invoking cora, including when another workflow calls it.
---

# Cora CLI

Cora reviews diffs and scans source using the configured LLM provider. Select the review scope explicitly; never stage or commit merely to make a review command work.

## Preconditions and safety

1. Check `cora --version`. This skill was checked against **0.15.0**. Use the installed subcommand help when versions differ or a flag is uncertain.
2. Before Git-based reviews, inspect:
   ```bash
   git rev-parse --show-toplevel
   git status --short
   git diff --stat
   git diff --cached --stat
   ```
   Run from the repository root. In short status, the first column is the index, the second is the working tree; `??` means untracked.
3. Respect existing provider/model configuration. Reviews can send diffs **and cross-file source context** to the configured endpoint and incur cost. A local CLI does not mean local LLM inference. Do not send secrets or restricted code to an unapproved provider.
4. Cora help can print live environment values, including credentials. For help only, omit credential variables from the child environment:
   ```bash
   env -u CORA_API_KEY -u GITHUB_TOKEN cora --help
   env -u CORA_API_KEY -u GITHUB_TOKEN cora review --help
   ```
   Apply the same prefix to other help calls; omit any additional credential variables exposed by future versions. Do not dump the environment or read `~/.cora/auth.toml` into tool output. Keep credentials available for actual authorized reviews.
5. A mention of Cora triggers this guidance, **not permission to execute every command**. A review request does not authorize staging, committing, pushing, amending, installing hooks, changing configuration, dismissing findings, uploading SARIF, or persisting findings to external memory.
6. Review commands may write caches, findings, and `.cora/history/`. Inspect status afterward and do not stage generated artifacts automatically.

## Choose the right scope

Use **one** scope flag per invocation. Do not rely on bare `cora review`: automatic fallback can select a different scope than intended.

| Situation | Command | Actual review input / caveat |
|---|---|---|
| Before staging, tracked edits | `cora review --unstaged` | `git diff`: working tree versus index; excludes untracked files and already-staged hunks. |
| After staging, before commit | `cora review --staged` | `git diff --cached`: exactly the staged snapshot, including staged new files; excludes later unstaged edits. |
| Mixed staged and unstaged work | Run each explicit mode separately | Use for separate snapshots; neither alone covers both. For the combined final state, see below. |
| Just committed | `cora review --commit HEAD` | Patch for the latest commit, independent of dirty working-tree edits. |
| A particular historical commit | `cora review --commit abc123` | Replace `abc123` with the requested ref/SHA. No checkout or amend needed. |
| Several committed changes | `cora review --commit 'HEAD~3..HEAD'` | Net diff between endpoints, not three independent reviews; requires sufficient history. |
| Committed, not pushed | `cora review --unpushed` | Patches from `git log -p '@{u}..HEAD'`; requires an upstream and excludes uncommitted changes. |
| Branch / PR review, even after pushing | `cora review --base origin/main` | `git diff origin/main...HEAD`: merge-base to HEAD, committed changes only. Replace with the actual target branch. |
| Existing patch / custom selection | `cora review --diff-file /tmp/change.patch` | Reviews the supplied diff, not the current Git staging area. |
| Whole project / no useful Git diff | `cora scan --path .` | File-content scan, not a patch review; respects supported extensions and exclusions. |

### Before staging: new files and combined work

Untracked files are absent from normal Git diffs. Never report them as reviewed by `--unstaged`.

- For new source files without changing the index, use a narrowly scoped scan, for example `cora scan --path . --include 'src/new_module.py'`. Confirm the file was actually included; ignored files and unsupported extensions can be skipped.
- If staging is explicitly authorized, stage only the selected files/hunks and review with `--staged`. Do not use `git add -A` or `git add -N` merely to expand review coverage.
- To review the combined final state of tracked staged and unstaged edits without touching the index, export a patch (requires an existing HEAD):
  ```bash
  patch_file=$(mktemp /tmp/cora-review.XXXXXX.patch)
  git diff --binary HEAD -- > "$patch_file"
  cora review --diff-file "$patch_file"
  review_status=$?
  rm -- "$patch_file"
  ```
  Preserve and report `review_status`. This still excludes untracked files; binary content is not meaningfully reviewed as source. Use a private temporary file and clean it up even on failure.
- Restrict a patch to selected paths with Git pathspecs, e.g. `git diff --cached -- packages/backend/`. **Do not use `cora review --include` or `--exclude`**: those flags belong to `scan`, not `review` in 0.15.0.

### After staging: preserve the reviewed snapshot

Run `cora review --staged`, verify findings against the code, and fix only what the user authorized. Edits made after staging are not in the staged review. With staging permission, re-stage only the corrected files/hunks and rerun `--staged`; preserve unrelated staged work and partial staging.

### After committing / before or after pushing

- For “review the commit I just made,” use `--commit HEAD`, not `--staged` or an automatic fallback.
- Before `--unpushed`, check the upstream and candidate commits:
  ```bash
  git rev-parse --abbrev-ref --symbolic-full-name '@{u}'
  git log --oneline '@{u}..HEAD'
  ```
- If no upstream exists, do not push or set one just for review. Use an explicit commit/range or the confirmed PR base. Ask if the intended scope cannot be inferred.
- Upstream tracking refs may be stale; fetch the appropriate remote when allowed and needed. Verify the target ref exists. Shallow CI clones may need more history to compute a merge base.
- After pushing, `--unpushed` can be empty. Use `--commit` for specific commits or `--base` for the branch's PR changes.
- For merge commits, ordinary `git show` may produce a combined/empty patch. If the intended scope is the merge relative to its first parent, use `cora review --commit 'HEAD~1..HEAD'`; confirm that comparison is intended.
- Fixing a historical finding does not authorize an amend, reset, rebase, or force-push. Review new working changes separately.

## Run, interpret, and verify

For agent-readable results:

```bash
cora review --staged --format json --quiet
```

For a local machine-readable artifact (choose an output path outside the reviewed tree):

```bash
cora review --base origin/main --format json --quiet --output-file /tmp/cora-review.json
```

- Supported formats: `pretty`, `json`, `compact`, `sarif`. Do not copy upstream examples using unsupported `--format markdown`.
- `--progress` emits NDJSON progress to **stderr**. Keep it separate from the result on stdout. Prefer non-streaming JSON for machine parsing; `--stream` is for interactive output.
- Keep auto-chunking enabled (default). There is `--no-auto-chunk`, not an `--auto-chunk` flag in 0.15.0. Narrow the scope or deliberately adjust `--max-diff-size` if necessary.
- Caching can reuse a prior diff review. Add `--no-cache` when a fresh LLM review is required, especially after changing review settings/context.
- `--severity major` filters lower-severity findings. Disclose filtering; do not describe filtered output as an exhaustive clean review.
- For valid JSON output, interpret the result fields directly. An empty `issues` array means the review found no issues. When it appears with `"should_block": false`, the review succeeded cleanly, even if `summary` is an empty string. Do not treat this result as an error or ask for a rerun just because the summary is blank:
  ```json
  {
    "issues": [],
    "should_block": false,
    "summary": ""
  }
  ```
- Exit **0 is not proof of no findings or complete coverage** by itself: warning-mode reviews, empty diffs, or skipped/failed analysis can return without blocking. For a valid, complete JSON result, `issues: []` is the evidence of no findings; still inspect reported scope and diagnostics for coverage gaps. A nonzero exit, invalid/truncated JSON, or explicit skipped/failed analysis requires investigation.
- `--ci` is documented to exit **2 if any findings** and skip the normal diff-size limit; quality-gate failure can also return 2. Treat other nonzero statuses as failures to investigate, not automatic evidence of code defects. Upstream exit-code tables are inconsistent; preserve the actual status and diagnostics.
- A scan can skip failed batches by default. Use `--no-continue-on-batch-error` when partial success is unacceptable. Lower `--batch-files` for provider limits; report skipped files/batches and incomplete reviews.
- Verify actionable findings against source and tests; LLM findings are not automatically correct. Do not blindly apply suggestions or dismiss findings to produce a passing report.
- After authorized fixes, run project tests/lint/typechecks and rerun the correct review scope. A committed diff does not include an uncommitted fix.
- Report: command and refs/scope, coverage gaps, exit status, actionable findings with file/line, and checks actually run. “No changes to review” means no review occurred, not a clean bill of health.

## CI and SARIF

With the actual target branch available locally and credentials supplied securely:

```bash
cora review --base origin/main --ci --format sarif --output-file /tmp/cora-results.sarif
```

Capture the exit status and retain the artifact even on findings; do not blindly hide failure with `|| true`. Generating SARIF is local; `cora review --upload` and `cora upload-sarif` publish to GitHub Code Scanning and require explicit upload authorization and appropriate credentials/permissions. `--upload` implies SARIF format.

Use trusted CI configuration and least-privilege tokens. Do not run untrusted PR code/config with secrets under `pull_request_target`. Cora can load project configuration and external prompts; do not assume a review-only job makes untrusted configuration safe.

## Configuration, authentication, and hooks

- Check authentication with `cora auth status`; have the user run interactive `cora auth login` if needed. For automation, use secret-managed `CORA_API_KEY` or supported provider-specific environment variables, never literal keys in commands or committed YAML.
- Settings priority: CLI flags → environment → project `.cora.yaml` → global `~/.cora/config.yaml` → auto-detection/defaults. `--config` selects another config file. Avoid provider/model overrides unless requested.
- `cora config validate` checks configuration. `cora config show` inspects resolved settings; inspect locally and redact sensitive values before sharing.
- Only when setup is requested: `cora init --no-hook` creates project config without installing a hook. Bare `cora init` **also installs a pre-commit hook**. Do not use `--force` to overwrite existing configuration without permission.
- `cora hook install` / `cora hook uninstall` change Git hooks. Inspect existing hooks first and preserve other tools' integration. Never bypass a hook or quality gate merely to finish a task.

## Committing with Cora (only when explicitly requested)

`cora commit` reviews the staged snapshot, generates a commit message, then offers accept/edit/abort. **It is a commit operation, not a review-only or message-preview command.**

- Stage only authorized changes and inspect the entire staged snapshot first; Cora will commit staged changes, including unrelated ones if left there.
- `--edit` opens the editor. `--yolo` removes confirmation and must not be used just because an agent cannot answer the interactive prompt; obtain authorization for noninteractive committing.
- `--no-review` skips review but **still proceeds toward committing**; it is not a safe message-only dry run.
- `--force` bypasses a failed quality gate. Do not use it without explicit bypass authorization.
- After an authorized commit, check `git log -1 --oneline` and `git status --short`. Do not push or open a PR without separate permission.

## Additional Cora workflows

Use only when relevant to the request, not as mandatory review prerequisites:

```bash
cora scan --path . --include 'src/**/*.ts' --exclude '**/generated/**'
cora scan --path . --incremental
cora index
cora explore 'authenticate'
cora brain 'authentication error handling'
cora callers authenticate
cora impact authenticate
cora trace authenticate
cora arch
cora affected src/auth.ts
cora findings list
cora findings stats
cora debt
```

Index before symbol/call-graph queries; refresh it after code changes. An index enriches review context but is not required for basic diff review. For affected tests, pass the files for the intended scope explicitly, or pipe `git diff --cached --name-only` to `cora affected --stdin` for staged files (ordinary newline-safe paths). Suggested tests do not replace the project's required checks.

`cora findings dismiss` / `reopen` mutate tracking state. `--memory` recalls from Uteke; `--learn` saves findings and implies `--memory`. Use these only when requested. `cora mcp` / `serve` launch long-running MCP servers; `cora install` changes agent configuration. Do not launch or install them for a normal CLI review.

## Sources and version caveats

Checked against installed `cora 0.15.0` help and upstream repository revision `7e7efe1a0cf69dc7383a8a9a50e402681b3741a8`:

- [Repository / README](https://github.com/codecoradev/cora-code)
- [CLI reference](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/docs/cli-reference.md)
- [Usage](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/docs/usage.md), [examples](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/docs/examples.md), [configuration](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/docs/configuration.md)
- [Exact Git diff semantics](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/src/git/diff.rs), [review selection and exit handling](https://github.com/codecoradev/cora-code/blob/7e7efe1a0cf69dc7383a8a9a50e402681b3741a8/src/commands/review.rs)

Some upstream examples disagree with current help (`cora scan .`, review include/exclude flags, Markdown output, default scope, exit codes). Prefer installed help for supported syntax and implementation/tests for behavior; do not propagate obsolete examples.
