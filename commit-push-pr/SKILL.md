---
name: commit-push-pr
description: Commit staged/working changes in an existing worktree with a Conventional Commit, push the branch to origin, and open a PR against the default branch. TRIGGERS - commit and push, open a PR, create pull request, ship this branch, commit push pr.
allowed-tools: Read, Bash
---

# Commit, Push, and Open a PR

## When to Use This Skill

Use this skill when working in an **existing** git worktree and the user asks to commit, push, and/or open a PR. Covers the full sequence end-to-end; also usable for any single step in isolation.

An explicit skill invocation (for example, `/skill:commit-push-pr`) is an affirmative request to execute the entire workflow: stage and commit the relevant changes, push the branch, and open the PR. Do not ask whether the user wants those actions or restate the plan for confirmation. Inspect the diff and make the ordinary workflow decisions yourself. Ask only when a real blocker or safety exception requires it, such as missing GitHub authentication or genuinely ambiguous unrelated changes.

Do **not** use this skill to create a new worktree — that's a separate concern. This assumes the worktree and branch already exist.

## Preconditions

```bash
git rev-parse --show-toplevel                 # 1. Confirm inside a git repo/worktree
git branch --show-current                      # 2. Identify the current branch
git status --short                              # 3. See what's changed
```

---

## 1. Stage and Commit

```bash
git status --short                              # 1. Review changed files
git diff                                        # 2. Review unstaged changes
git add <specific files>                        # 3. Stage only relevant files (avoid `git add -A` unless confirmed)
git commit -m "<type>(<scope>): <description>"  # 4. Conventional Commit, imperative mood, no trailing period
```

**Conventional Commit types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`.

Only stage files relevant to the change. If unsure whether a file belongs, ask the user rather than guessing.

---

## 2. Push the Branch

```bash
git branch --show-current                              # 1. Confirm branch name
git push -u origin "$(git branch --show-current)"       # 2. Push and set upstream (first push)
git push                                                 # 2b. Subsequent pushes once upstream is set
```

Use `-u`/`--set-upstream` only on the first push of a new branch; a plain `git push` suffices afterward since upstream is already tracked.

---

## 3. Detect the Default Branch

```bash
git remote show origin | sed -n '/HEAD branch/s/.*: //p'   # 1. Query remote's default branch (usually main)
```

Fall back to `main` if this doesn't return a value, but prefer the detected result.

---

## 4. Open the PR

```bash
gh pr create --base <default-branch> --head "$(git branch --show-current)" \
  --title "<Conventional Commit style title>" \
  --body "<PR body, see structure below>"
```

### PR title
- Mirror the commit's Conventional Commit subject (e.g. `feat(client): add request ID correlation`).
- Keep it under ~72 chars; no trailing period.

### PR body — write a structured summary, not one-liners

Aim for a body that lets a reviewer (or future you, scanning the PR list) understand the **why** and the **shape** of the change without opening the diff. Use the diff (`git diff <default-branch>...HEAD`) and the commit log (`git log <default-branch>..HEAD --oneline`) as sources — never invent behavior that isn't in the diff.

**Recommended structure** (in this order; omit sections that genuinely don't apply, e.g. a docs-only PR may not need Testing):

#### Summary
One short paragraph (2–4 sentences) stating what changed and the user-facing or developer-facing impact. Lead with the outcome, not the implementation.

> Stream Playground requests through Deep Chat's OpenAI-compatible transport so the existing Playground UI can be reused against any provider behind that shim, without a separate streaming path.

#### Why / Context (optional but encouraged)
The motivation behind the change — the problem, the constraint, or the gap. Skip if the commit message already says it and the change is self-evident.

> The previous Playground was hard-wired to one provider's streaming format, so adding a new provider meant duplicating the request/response handling. Routing through the OpenAI-compatible layer lets new providers plug in by configuration alone.

#### Changes
A short bulleted list of the meaningful changes. Each bullet should name the area/module and what it now does, not just restate the commit subject. 3–7 bullets is a good target; collapse trivial ones.

- **Client**: thread a generic `X-Request-ID` header through every request and surface it on responses so logs across services can be correlated.
- **API**: add a provider picker to the Playground header; selection is persisted in the URL so links are shareable.
- **Tooling**: include two sample browser tools (read tab URL, capture screenshot) wired into the existing tool-call detail panel.
- **UI**: render tool-call args and results inline beneath assistant messages; previously they only appeared in the raw event log.

#### Testing
Describe what was exercised **beyond the pre-commit gates** (`bun run test` / `typecheck` / lint are assumed — don't list them). Examples of useful content here: manual browser flows, edge cases hit, fixtures added, screenshots attached, known gaps.

- Manually verified Playground streaming against three providers end-to-end (request → streamed tokens → final usage event).
- Confirmed tool-call detail panel renders both argument and result payloads for built-in and sample browser tools.
- Added fixtures for the usage lookup path; existing snapshot tests cover the streaming shape.

#### Notes / Follow-ups (optional)
Anything a reviewer should know that's not obvious from the diff: breaking changes, deprecations, TODOs left intentionally, related issues/PRs.

### Multi-commit PRs
When `git log <default-branch>..HEAD --oneline` shows more than one commit, weave them into the **Changes** bullets in logical order (not commit order). Squash or reorder commits first if the history is noisy and the user agrees.

### `--body` formatting
Pass the body via a heredoc to `gh pr create --body-file -` so Markdown (headings, bullets, code spans) renders correctly in GitHub — escaping bullets through `--body` directly is fragile.

```bash
gh pr create --base <default-branch> --head "$(git branch --show-current)" \
  --title "feat(playground): stream via Deep Chat OpenAI-compatible transport" \
  --body-file - <<'EOF'
## Summary
...
EOF
```

### Auth
- If `gh` isn't authenticated (`gh auth status` fails), stop and tell the user rather than attempting a workaround.

---

## 5. Validation (SLO)

```bash
git log -1 --oneline                            # 1. Confirm commit landed
git status --short                              # 2. Confirm clean tree, no stray unstaged files
gh pr view --web=false                           # 3. Confirm PR was created and view its URL/status
```

---

## Guardrails

- Never commit, push, or open a PR unless the user explicitly authorized it in the current turn. An explicit `/skill:commit-push-pr` invocation authorizes all three actions together; execute the full sequence without a follow-up confirmation. For a natural-language request that names only a subset of actions, perform only that requested subset.
- **Never** force-push.
- Stage only relevant files — no unrelated changes bundled into the commit.

---

## Troubleshooting

| Issue                              | Cause                                   | Solution                                                        |
| ----------------------------------- | ---------------------------------------- | ----------------------------------------------------------------|
| `gh: command not found`             | GitHub CLI not installed                 | Ask user to install `gh`, or push only and share compare URL     |
| `gh auth status` fails               | Not logged into `gh`                     | Ask user to run `gh auth login`; don't attempt a workaround      |
| Push rejected (non-fast-forward)     | Remote branch has diverged               | `git pull --rebase`, resolve conflicts, then re-push — ask first |
| No upstream set                     | First push of new branch                 | Use `git push -u origin <branch>`                                |
