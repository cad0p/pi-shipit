# pi-shipit

Quality gates for shipping code with the [Pi coding agent](https://pi.dev/).

Two human-in-the-loop skills that use subagents for thorough review and safe upstream contribution.

## Skills

### `review-fix-loop`

Iterative code review with parent triage. Delegate to a reviewer subagent, triage findings yourself, dispatch approved fixes to a worker subagent, and repeat until converged.

**When to use:**
- Before shipping any non-trivial change
- Addressing reviewer feedback (human or AI)
- Quality assurance passes

**Invoke:** `/skill:review-fix-loop`

### `submit-fork-pr`

Two-stage fork-to-upstream PR workflow. Create an internal PR on your fork first, run it through Copilot review and CI, merge, then open a clean upstream draft PR.

**Why two stages?** Queue multiple fixes on your fork while the upstream maintainer works through approvals. Each fix gets its own review gate before reaching the upstream repo.

**When to use:**
- Contributing to upstream repos via a personal fork
- You want AI pre-review and CI greenlight before human review

**Invoke:** `/skill:submit-fork-pr`

## Install

**Via npm (recommended):**
```bash
pi install npm:pi-shipit
```

**Via git:**
```bash
pi install git:github.com/cad0p/pi-shipit
```

Requires [pi-subagents](https://github.com/nicobailon/pi-subagents) for the `subagent` tool.

## License

MIT
