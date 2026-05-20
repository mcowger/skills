---
name: push-to-pr
description: Push local commits directly to a contributor's fork and branch from a PR, without adding a git remote. Use when pushing changes to a PR branch, pushing fixes into someone else's PR, updating a pull request with local edits, or when the user says "push to the PR", "push to this PR's branch", "push my changes to the contributor's fork", or "update the PR with my local commits". Also use when the user is on a branch associated with an open PR and asks to push, especially when the branch tracks a fork rather than origin. Prefer this over manually adding remotes.
compatibility: Requires git and gh (GitHub CLI) with authenticated access
---

# Push to PR

When you're working on a branch tied to an open pull request and need to push local commits back to the contributor's fork, the usual workflow is to add their fork as a remote, fetch, and push. That's tedious and leaves stale remotes lying around. This skill pushes directly to the contributor's fork using a one-shot URL — no remote setup, no cleanup.

## When to use this

You're on a local branch, there's an associated PR, and you need to push commits to the PR author's fork/branch. This is common when:

- A maintainer is pushing fixes or tweaks into a contributor's PR
- You've made local edits on a PR branch and want to update the PR
- You want to push to a fork without polluting your remote list

## Workflow

### Step 0: Identify the current branch and find the associated PR

First, get the current branch name, then look up the associated PR number:

```bash
BRANCH=$(git branch --show-current)
gh pr list --head "$BRANCH" --state open --json number --jq '.[0].number'
```

If the branch name alone doesn't find the PR (e.g., the PR was opened from a fork where the branch name differs from the local one), try searching by the current commit:

```bash
gh pr list --state open --search "$(git rev-parse HEAD)" --json number --jq '.[0].number'
```

If the user already knows the PR number, skip this step.

### Step 1: Retrieve the contributor's fork and branch

Use `gh pr view` to get the head repository (the fork) and head ref (the branch) from the PR:

```bash
gh pr view <PR_NUMBER> --json headRepositoryOwner,headRepository,headRefName --jq '"\(.headRepositoryOwner.login) \(.headRepository.name) \(.headRefName)"'
```

This returns three values:
- **CONTRIBUTOR_NAME** — the GitHub username of the PR author (owner of the fork)
- **REPO_NAME** — the name of the fork repository
- **CONTRIBUTOR_BRANCH** — the branch name in the contributor's fork

If `headRepository` is null (e.g., the fork was deleted), the push will fail — inform the user that the contributor's fork no longer exists.

### Step 2: Construct and execute the push command

Push directly to the contributor's fork using a one-shot HTTPS URL. No remote needed:

```bash
git push https://github.com/<CONTRIBUTOR_NAME>/<REPO_NAME>.git HEAD:<CONTRIBUTOR_BRANCH>
```

This pushes your current HEAD to the contributor's branch on their fork. The URL is used inline — git handles authentication through your configured credential helper, and no remote is created or modified.

### Step 3: Confirm

After the push succeeds, confirm to the user what was pushed and where:

```
Pushed to <CONTRIBUTOR_NAME>/<REPO_NAME>:<CONTRIBUTOR_BRANCH> (PR #<PR_NUMBER>)
```

## Edge cases

- **Same repository PRs**: If the PR head is the same repo (not a fork), the push URL uses the current repo's origin. In this case, a simple `git push origin HEAD:<BRANCH>` may be sufficient. Check whether `headRepositoryOwner.login` matches the base repo owner.
- **Force push**: If the local branch has been rebased or rewritten, you may need `--force-with-lease`. Always prefer `--force-with-lease` over `--force` — it's safer because it rejects the push if the remote has changes you haven't seen. Only use it when the user explicitly asks or when you know the history has been rewritten.
- **Push rejection**: If the push is rejected because the remote has diverged, inform the user. They may need to pull or rebase first. Do not force-push without asking.
- **Deleted fork**: If the contributor deleted their fork after opening the PR, the push will fail. Inform the user — there's no way to push to a deleted fork.
- **Permissions**: You need write access to the contributor's fork (or maintainer permissions on the upstream repo) for this to work. If authentication fails, inform the user.

## Full example

A maintainer wants to push a typo fix into PR #42:

```bash
# Step 0: Already know PR number is 42

# Step 1: Get contributor info
gh pr view 42 --json headRepositoryOwner,headRepository,headRefName --jq '"\(.headRepositoryOwner.login) \(.headRepository.name) \(.headRefName)"'
# Output: janedev my-project fix-typo-bug

# Step 2: Push directly to the fork
git push https://github.com/janedev/my-project.git HEAD:fix-typo-bug

# Step 3: Confirm
# → Pushed to janedev/my-project:fix-typo-bug (PR #42)
```

No remote added, no cleanup needed.
