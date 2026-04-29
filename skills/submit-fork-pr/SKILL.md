---
name: submit-fork-pr
description: Two-stage fork-to-upstream PR workflow with automated review gates. Use when contributing via a personal fork and want Copilot pre-review + CI greenlight before upstream human review. Internal fork PR first, then upstream draft PR.
---

# Submit Fork PR with Copilot Pre-Review

Ship changes from a fork through Copilot review gates before marking ready for human review.

> **Why fork → internal PR → upstream PR?** This two-stage pattern lets you queue multiple independent fixes on your fork while the upstream maintainer works through approvals. Each fix gets its own Copilot review and CI run before it ever reaches the upstream repo. When you create the upstream PR, always base it off the upstream's `main` (or default) branch — not your fork's `main` — so the maintainer sees a clean, linear diff.

## Prerequisites
- **Code has been through at least one `review-fix-loop` cycle** — internal review and fixes are done before any PR is created
- Fork exists on GitHub (e.g. `<your-fork>/<repo>`)
- `gh` CLI authenticated with `repo` scope

## Workflow

### 1. Push branch to fork

```bash
subagent({
  agent: "worker",
  cwd: "~/Documents/GitHub/<your-fork>/<repo>",
  task: `
    git checkout -b feature/xyz
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

### 2. Create draft PR on fork

```bash
gh pr create --repo <FORK> --base main --head feature/xyz \
  --draft --title "feat: ..." --body "..."
```

### 3. Request Copilot review explicitly

**By default, Copilot does NOT auto-review PRs.** You must request it manually, like any human reviewer:

```bash
# Request Copilot as a reviewer
gh pr edit --repo <FORK> <NUM> --add-reviewer "copilot"
```

If the repo has a **ruleset** with "Automatically request Copilot code review" enabled, this step is unnecessary — but explicitly requesting it is harmless and ensures it runs.

### 4. Poll for Copilot comments

```bash
for i in {1..5}; do
  COMMENTS=$(gh api repos/<FORK>/pulls/<NUM>/reviews --jq 'length')
  [ "$COMMENTS" -gt 0 ] && echo "Copilot reviewed" && break
  echo "Waiting for Copilot... ($i/5)"
  sleep 60
done
```

### 5. Address comments (mandatory if present)

- Read each comment
- Fix or decline (explain reasoning in reply)
- Spawn subagent with `cwd: <fork>` to apply fixes
- Amend commit and force push
- Reply to each comment and mark resolved
- Re-request Copilot review: `gh pr edit --repo <FORK> <NUM> --add-reviewer "copilot"`
- Go back to step 4 and poll again

Only proceed when comments are resolved or you've hit rate-limit twice.

### 6. Merge fork PR

```bash
gh pr merge --repo <FORK> <NUM> --squash --delete-branch
```

### 7. Create upstream draft PR

```bash
gh pr create --repo <UPSTREAM> --base main --head <FORK_OWNER>:main \
  --draft --title "feat: ..." --body "..."
```

### 8. Request upstream Copilot review

```bash
gh pr edit --repo <UPSTREAM> <NUM> --add-reviewer "copilot"
```

### 9. Poll for CI and Copilot in parallel

Both must converge before step 11.

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

### 10. Address failures from either check

- Fix on fork (subagent with `cwd: <fork>`)
- Merge to fork `main`
- Upstream draft PR auto-updates
- Reply to Copilot comments and mark resolved
- Re-request Copilot review on upstream PR
- Go back to step 9

### 11. Mark as ready for review

Only after CI is green AND Copilot is clean:
```bash
gh pr ready --repo <UPSTREAM> <NUM>
```

### 12. Optional: final Copilot round

Wait 5 minutes after marking ready, poll for late comments. Address if needed.

## Rate-Limit Handling
- If Copilot rate-limits, note it in the PR body and proceed after one retry.
- Do not block indefinitely.

## Tips
- Use the `review-fix-loop` skill for addressing Copilot comments iteratively.
- The `cwd` trick lets subagents do native git; use `gh api` only as fallback.
- `gh pr edit --add-reviewer "copilot"` is the explicit trigger. Do not assume auto-review.
