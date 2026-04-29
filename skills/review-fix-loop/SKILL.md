---
name: review-fix-loop
description: Iterative code review using subagents with parent triage. Use for thorough code review, addressing reviewer feedback, or quality assurance before shipping. Runs reviewer→triage→worker→verify cycles until LGTM or all issues declined.
---

# Review → Fix → Review Loop

Use subagents to iteratively review and fix code until it converges.

## Setup

Point subagents at the target repo with `cwd`:
```
subagent({ agent: "reviewer", cwd: "<repo-path>", task: "..." })
```

## Loop

Run this cycle until the reviewer outputs **LGTM**, or until all remaining issues are explicitly triaged as "decline with explanation":

### 1. Run reviewer
```
subagent({
  agent: "reviewer",
  cwd: "<repo-path>",
  task: "Review <file(s)>. Output a numbered list of issues with concrete fixes, or LGTM if clean."
})
```

### 2. Triage by parent (you)
For each issue raised:
- **Approve for fix** → include in the worker task
- **Decline** → note the reason (out of scope, intentional trade-off, false positive, etc.)

Do not blindly hand every issue to the worker. The parent acts as the triager.

### 3. Run worker on approved fixes only
```
subagent({
  agent: "worker",
  cwd: "<repo-path>",
  task: `Apply these fixes to <file(s)>: <paste approved subset>. Keep changes minimal.`
})
```

### 4. Re-run reviewer for verification
Repeat the cycle. If the reviewer re-raises a previously declined issue, reference the prior decision and re-confirm.

### 5. Converge and report
Stop when either:
- Reviewer outputs **LGTM** with no issues, **or**
- All remaining issues are explicitly declined

Provide a final summary to the user:
- What was fixed and why
- What was declined and why

## Tips

- Keep review tasks narrow (one file or one function at a time).
- If subagents fail due to sandbox/CWD issues, spawn them with explicit `cwd`.
- If your subagent model differs from your main model, set it per-run via settings override or agent config.
- Save converged diffs with `git diff` so you don't lose work.
