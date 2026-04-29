---
name: submit-fork-pr
description: Cross-fork PR workflow. Use when creating a new upstream PR or updating an existing one from a personal fork.
---

# Submit Fork PR with Copilot Pre-Review

Ship changes from a personal fork to an upstream repo.

## Fork structure

| Branch | Purpose |
|---|---|
| `origin/main` | Personal integration: `upstream/main` + your merged features |
| `feature/*` | Review + upstream submission branch |

Your fork's `main` is your integration branch. Before starting new work, always rebase it onto `upstream/main` to pick up the latest upstream changes. Feature branches branch from `origin/main` and are rebased onto `upstream/main` for upstream submission. Upstream PRs come from feature branches, never from `origin/main`.

## Prerequisites
- **Code has been through at least one `review-fix-loop` cycle** — internal review and fixes are done before any PR is created
- Fork exists on GitHub (e.g. `<your-fork>/<repo>`)
- `gh` CLI authenticated with `repo` scope

## Workflow

### 1. Sync integration branch

```bash
git fetch upstream
git checkout main
git rebase upstream/main
git push --force origin main
```

### 2. Push branch to fork

```bash
subagent({
  agent: "worker",
  cwd: "~/Documents/GitHub/<your-fork>/<repo>",
  task: `
    git checkout -b feature/xyz origin/main
    git add .
    git commit -m "feat: ..."
    git push -u origin feature/xyz
  `
})
```

**Fallback** (if sandbox blocks `.git/`):
```bash
BASE_SHA=$(gh api repos/<FORK>/branches/main --jq '.commit.sha')
gh api repos/<FORK>/git/refs -X POST \
  -f ref="refs/heads/feature/xyz" -f sha="$BASE_SHA"
# create blobs → tree → commit → update ref
```

### 3. Create draft PR on fork

```bash
gh pr create --repo <FORK> --base main --head feature/xyz \
  --draft --title "feat: ..." --body "..."
```

### 4. Request Copilot review explicitly

**By default, Copilot does NOT auto-review PRs.** You must request it manually, like any human reviewer:

```bash
# Request Copilot as a reviewer
gh pr edit --repo <FORK> <NUM> --add-reviewer "copilot"
```

If the repo has a **ruleset** with "Automatically request Copilot code review" enabled, this step is unnecessary — but explicitly requesting it is harmless and ensures it runs.

### 5. Poll for Copilot comments

```bash
for i in {1..5}; do
  COMMENTS=$(gh api repos/<FORK>/pulls/<NUM>/reviews --jq 'length')
  [ "$COMMENTS" -gt 0 ] && echo "Copilot reviewed" && break
  echo "Waiting for Copilot... ($i/5)"
  sleep 60
done
```

### 6. Address comments (mandatory if present)

- Read each comment
- Fix or decline (explain reasoning in reply)
- Spawn subagent with `cwd: <fork>` to apply fixes
- Amend commit and force push
- Reply to each comment and mark resolved
- Re-request Copilot review: `gh pr edit --repo <FORK> <NUM> --add-reviewer "copilot"`
- Go back to step 5 and poll again

Only proceed when comments are resolved or you've hit rate-limit twice.

### 7. Merge fork PR

```bash
gh pr merge --repo <FORK> <NUM> --squash --delete-branch
```

### 8. Prepare upstream submission branch

After the internal PR is squash-merged to `origin/main`, prepare the feature branch for upstream submission:

```bash
git checkout feature/xyz
git fetch upstream
git rebase upstream/main
# resolve conflicts if any; then:
git push --force origin feature/xyz
```

> If you prefer not to integrate into `origin/main` first, skip the internal PR and just rebase the feature branch onto `upstream/main` directly.

### 9. Create upstream draft PR

```bash
gh pr create --repo <UPSTREAM> --base main --head <FORK_OWNER>:feature/xyz \
  --draft --title "feat: ..." --body "..."
```

### 10. Request upstream Copilot review

```bash
gh pr edit --repo <UPSTREAM> <NUM> --add-reviewer "copilot"
```

### 11. Poll for CI and Copilot in parallel

Both must converge before step 13.

**CI:**
```bash
gh pr checks --repo <UPSTREAM> <NUM> --watch --interval 10
```

**Copilot:**
```bash
for i in {1..5}; do
  COMMENTS=$(gh api repos/<UPSTREAM>/pulls/<NUM>/reviews --jq 'length')
  [ "$COMMENTS" -gt 0 ] && echo "Copilot reviewed" && break
  echo "Waiting... ($i/5)"
  sleep 60
done
```

### 12. Address failures from either check

- Fix on the feature branch (subagent with `cwd: <fork>`)
- Rebase onto `upstream/main`:
  ```bash
  git checkout feature/xyz
  git fetch upstream
  git rebase upstream/main
  git push --force origin feature/xyz
  ```
- Upstream draft PR auto-updates
- Reply to Copilot comments and mark resolved
- Re-request Copilot review on upstream PR
- Go back to step 11

> If you also want the fix in your `origin/main` integration branch, merge or cherry-pick it there separately. This is optional and independent of the upstream iteration.

### 13. Mark as ready for review

Only after CI is green AND Copilot is clean:
```bash
gh pr ready --repo <UPSTREAM> <NUM>
```

### 14. Optional: final Copilot round

Wait 5 minutes after marking ready, poll for late comments. Address if needed.

## Parallel PRs

Feature branches are independent. Upstream PRs for `feature/A` and `feature/B` coexist naturally because each is rebased onto `upstream/main`.

When upstream merges one of your PRs, update your integration branch and other open branches:

```bash
git fetch upstream

# Update integration branch
git checkout main
git rebase upstream/main
git push --force origin main

# For each still-open upstream feature branch
git checkout feature/abc
git rebase upstream/main
git push --force origin feature/abc
```

For stacked PRs (feature B depends on feature A), note the dependency in the PR body. After A merges upstream, rebase B onto `upstream/main`.

## Rate-Limit Handling
- If Copilot rate-limits, note it in the PR body and proceed after one retry.
- Do not block indefinitely.

## Tips
- Use the `review-fix-loop` skill for addressing Copilot comments iteratively.
- The `cwd` trick lets subagents do native git; use `gh api` only as fallback.
- `gh pr edit --add-reviewer "copilot"` is the explicit trigger. Do not assume auto-review.