# Swarm — Parallel Coding Sprint Skill

Parallel multi-agent coding sprints using git worktree isolation.
Inspired by Anthropic's leaked COORDINATOR_MODE architecture.

## When to Use This Skill

Use swarm when asked to do 2 or more coding tasks on a repository.

Default behavior:
- 1 task → do it directly, no swarm needed
- 2+ tasks touching different parts of the codebase → use swarm
- 2+ tasks ALL touching schema/auth → serialize them, still use swarm but one group at a time

Trigger phrases:
- "Run a swarm sprint: task1, task2, task3"
- "Parallel sprint on [repo]"
- "Use swarm for these tasks"
- Any job with 2+ separable coding tasks

## The Golden Rule — ALWAYS Plan First

**Before spawning any agents, you MUST run the conflict analyzer.**

```bash
node <workspace>/skills/swarm/swarm.js --repo <repo_path> --tasks tasks.json --plan-only
```

Read the output:
- `✓ No conflicts` → all tasks run in parallel, one group
- `⚠ HIGH conflict` → affected tasks are auto-serialized into separate groups
- `LOW conflict` → runs parallel, watch the merge

**Never skip this step.** Two agents modifying the same file = merge conflict = wasted work.

## Full Workflow

### Step 1 — Write tasks.json

Create a tasks file. Be specific. Vague tasks produce wandering agents.

```json
[
  {
    "id": "short-unique-id",
    "description": "Exactly what to build or fix — be specific",
    "role": "coder",
    "successCriteria": [
      "Specific verifiable outcome",
      "TypeScript compiles clean"
    ]
  }
]
```

### Step 2 — Run conflict analysis (mandatory)

```bash
node <workspace>/skills/swarm/swarm.js --repo <repo_path> --tasks tasks.json --plan-only
```

### Step 3 — Create worktrees

```bash
node <workspace>/skills/swarm/swarm.js --repo <repo_path> --tasks tasks.json
```

This creates one isolated git worktree + branch per task and writes `swarm-packages.json`.

### Step 4 — Spawn subagents

Read `swarm-packages.json`. For each package, spawn a subagent with:
- Working directory set to `worktreePath`
- The `instructions` field as the task prompt
- Instruction to stay inside that directory only
- Instruction to report back: STATUS / FILES_CHANGED / SUMMARY / BLOCKERS

### Step 5 — Review each agent's output

```bash
git diff main..swarm/<branch-name>
```

Accept → proceed to merge
Reject → note the issue, log it, skip merge

### Step 6 — Merge passing work

```bash
git merge swarm/<branch-name>
```

Merge one at a time. Verify TypeScript is clean after each merge.

### Step 7 — Cleanup (always, no exceptions)

```bash
node <workspace>/skills/swarm/swarm.js --repo <repo_path> --cleanup swarm-packages.json
```

This deletes all worktrees and branches — merged or rejected. Always run this.

## Agent Roles

| Role | Behavior |
|------|---------|
| `coder` | Implements the task. Does not refactor unrelated code. Runs tsc when done. |
| `reviewer` | Reviews the coder's diff skeptically. Flags bugs, type errors, missing error handling. Does not approve weak work. |
| `tester` | Writes tests for the new code. Follows existing test patterns. |

## Script Location

The script path depends on where the skill is installed. Use the workspace-relative path:

```
<workspace>/skills/swarm/swarm.js
```

To find your workspace path: check `TOOLS.md` or run `openclaw status`.

## High-Conflict File Areas

Always serialize (never parallelize) tasks that both touch:
- Database schema files (`schema.prisma`, migration SQL)
- Auth/session middleware
- Main router or index registration files
- Shared config or environment files

## Rules

- Planning step is mandatory — no exceptions
- Worktrees are always deleted after sprint — no exceptions
- Never push from a worktree — coordinator handles all git operations
- Reviewer role should always use a cheaper model (Haiku) — coder uses Sonnet
- Max 5 parallel agents — review overhead outweighs gains beyond that
- Sprint log written to `memory/swarm-log.md` after every run

## Dry Run

Always available for safe preview:
```bash
node <workspace>/skills/swarm/swarm.js --repo <repo_path> --tasks tasks.json --dry-run
```
