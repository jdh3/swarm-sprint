# Swarm Sprint 🐝

**Parallel coding sprints for OpenClaw agents — with git worktree isolation, conflict planning, and automatic cleanup.**

Inspired by the multi-agent coordinator architecture revealed in Anthropic's leaked Claude Code source (March 2026).

---

## Why This Is Different From Just "Spinning Up Agents"

Most people who want parallel AI coding just tell their agent to do multiple things at once. Here's what actually happens:

```
❌ Naive parallel agents (broken):
Agent 1 edits payroll.ts ──┐
Agent 2 also edits payroll.ts ──┘
→ Last write wins. One agent's work is silently overwritten.
→ Merge conflicts. Broken builds. Wasted tokens.
```

Swarm solves this with **git worktree isolation** — the same approach Anthropic built into Claude Code:

```
✅ Swarm (safe parallel):
Agent 1 → own folder + own branch → edits payroll.ts safely
Agent 2 → own folder + own branch → edits invoices.ts safely
→ Coordinator reviews each diff independently
→ Merges passing work, rejects failing work
→ Deletes all worktrees when done
```

Each agent gets its own private copy of the repo. They cannot touch each other's files. The coordinator reviews the output before anything lands on main.

**The other thing that makes this different:** a mandatory conflict planning step runs before any agent is spawned. It analyzes your task list, detects which tasks would touch the same files, and flags or re-sequences them automatically. You don't discover the conflict after an hour of wasted work — you see it in 2 seconds before anything starts.

---

## What You Get

- ✅ True parallel execution — 3 tasks in the time of 1
- ✅ Zero file conflicts — git worktree isolation per agent
- ✅ Conflict analysis before spawning — catches problems upfront
- ✅ Typed agent roles — coder, reviewer, tester
- ✅ Coordinator review gate — bad work doesn't land on main
- ✅ Auto-cleanup — worktrees always deleted, no disk waste
- ✅ Sprint log — every run logged to `memory/swarm-log.md`
- ✅ Works on any codebase — not just TypeScript or any specific stack

---

## Requirements

- **Node.js** — any version 16+
- **Git** — version 2.5+ (worktree support)
- **OpenClaw** — installed and running
- **A git repository** — the codebase your agent will work on

---

## Installation

### Step 1 — Find your workspace path

Your OpenClaw workspace is where your agent's files live. The default is:
```
~/.openclaw/workspace/
```

If you set up a named profile or agent, it may be:
```
~/.openclaw/workspace-<name>/
```

Not sure? Run: `openclaw status` and look for the workspace path.

### Step 2 — Copy the skill folder

```bash
# Default workspace
cp -r swarm/ ~/.openclaw/workspace/skills/swarm/

# Named workspace (replace <name> with yours)
cp -r swarm/ ~/.openclaw/workspace-<name>/skills/swarm/
```

Your workspace should now look like:
```
~/.openclaw/workspace/
  skills/
    swarm/
      swarm.js            ← the script your agent runs
      SKILL.md            ← agent instructions (don't edit this)
      README.md           ← this file
      example-tasks.json  ← template to copy from
```

### Step 3 — Activate it

Tell your agent once:
> "I installed the swarm skill. Use it by default whenever I give you 2 or more coding tasks."

After that it's automatic. The agent reads `SKILL.md` on every session and knows to use swarm for multi-task sprints.

---

## How to Use It

### 1. Create a tasks.json file

In your workspace, create a file describing the tasks. Be specific — vague tasks produce wandering agents.

```json
[
  {
    "id": "fix-auth-timeout",
    "description": "Fix session timeout — users get logged out after 1 hour even while active. Add silent token refresh to the auth middleware so active sessions stay alive.",
    "role": "coder",
    "successCriteria": [
      "Active sessions no longer expire mid-use",
      "Refresh happens silently without user action",
      "TypeScript compiles with no errors"
    ]
  },
  {
    "id": "build-settings-page",
    "description": "Build an Admin Settings page at /settings — must include: notification preferences, timezone selector, and a save button that persists to the database.",
    "role": "coder",
    "successCriteria": [
      "Page loads at /settings without errors",
      "Settings save and reload correctly",
      "TypeScript compiles with no errors"
    ]
  },
  {
    "id": "write-invoice-tests",
    "description": "Write end-to-end tests for the invoice workflow — create invoice, edit line items, mark as paid. Use existing test patterns in the codebase.",
    "role": "tester",
    "successCriteria": [
      "Tests cover create, edit, and paid flow",
      "Tests pass locally",
      "Follows existing test file naming and structure"
    ]
  }
]
```

### 2. Tell your agent to run it

> "Run a swarm sprint using tasks.json on my repo"

Or just list the tasks directly:
> "Run a swarm sprint: fix the auth timeout, build the settings page, write invoice tests"

### 3. What happens next (automatic)

```
1. PLAN    — conflict analyzer checks all tasks for file overlap
             HIGH conflicts → flagged and serialized automatically
             LOW conflicts → noted, safe to run in parallel

2. CREATE  — one git worktree + branch created per task
             e.g. /my-repo-swarm-fix-auth-timeout  [branch: swarm/...]
                  /my-repo-swarm-build-settings-page [branch: swarm/...]

3. SPAWN   — one subagent per task, each told:
             - "Your working directory is /my-repo-swarm-<task>"
             - "Stay in that directory only"
             - "Report back with STATUS, FILES_CHANGED, SUMMARY"

4. REVIEW  — coordinator reads each agent's diff
             Passes → queued for merge
             Fails  → rejected, noted in sprint log

5. MERGE   — passing branches merged into main one by one

6. CLEANUP — ALL worktrees and branches deleted
             Merged or rejected — everything goes
             No disk waste, no leftover branches
```

### 4. Manual commands (if you prefer direct control)

```bash
# Conflict check only — no changes made
node skills/swarm/swarm.js --repo /path/to/repo --tasks tasks.json --plan-only

# Full dry run — see exactly what would happen
node skills/swarm/swarm.js --repo /path/to/repo --tasks tasks.json --dry-run

# Create worktrees and generate agent packages
node skills/swarm/swarm.js --repo /path/to/repo --tasks tasks.json

# Cleanup after agents finish
node skills/swarm/swarm.js --repo /path/to/repo --cleanup swarm-packages.json
```

---

## Agent Roles

Assign each task a role to get the right behavior:

| Role | What it does | When to use |
|------|-------------|-------------|
| `coder` | Implements the task, stays in scope, verifies TypeScript | Default — use for any build/fix task |
| `reviewer` | Reads the coder's diff skeptically, flags bugs, won't rubber-stamp | After a complex coder task |
| `tester` | Writes tests for the code, follows existing patterns | When you want test coverage added |

**Pro tip:** For important features, run a `coder` task first, then a `reviewer` task pointing at the same output. The reviewer sees it with fresh eyes.

---

## Task Writing Tips

**Good tasks are:**
- Specific enough that success is obvious
- Small enough for one agent to finish in one session
- Touching a distinct area of the codebase

**Bad tasks look like:**
- "Fix the backend" — too vague, agent will wander
- "Improve performance" — no verifiable end state
- Two tasks both needing database schema changes — HIGH conflict, serialize them

**High-conflict areas (serialize these, don't parallelize):**
- Database schema files (`schema.prisma`, migration files)
- Auth/session middleware
- Shared config files
- Main router/index files

---

## Conflict Analysis Output

When you run `--plan-only`, you'll see output like:

```
✓ No file conflicts detected — all tasks can run fully in parallel
```
or
```
⚠ HIGH conflict: Task "add-user-roles" and "fix-auth-flow" both touch [auth, schema]
  → Recommend serializing these tasks
```
or
```
LOW conflict: Task "build-settings" and "write-tests" share [frontend]
  → Likely fine but watch the merge
```

HIGH = automatically serialized into separate groups
LOW = runs in parallel, coordinator watches the merge carefully

---

## Sprint Log

Every sprint appends a one-line summary to your workspace's `memory/swarm-log.md`:

```
## Swarm Sprint — 2026-04-02T20:45:00Z
- Repo: /my-repo
- Tasks: 3
- Results:
  - fix-auth-timeout: MERGED — Refresh token middleware added, 3 files changed
  - build-settings-page: MERGED — Settings page built at /settings, 5 files changed
  - write-invoice-tests: REJECTED — Tests failing, serialization issue in test setup
```

No disk cost. Just a clean record of every sprint.

---

## Files

| File | Purpose |
|------|---------|
| `swarm.js` | The script — conflict analysis, worktree management, cleanup |
| `SKILL.md` | Agent instructions — OpenClaw reads this automatically |
| `README.md` | This file |
| `example-tasks.json` | Copy this and edit it to start your first sprint |

---

## Frequently Asked Questions

**Do I need to do anything after installing?**
Just tell your agent once that the skill is installed. After that it uses it automatically for 2+ task jobs.

**What if two tasks conflict?**
The planner catches it before anything runs. HIGH conflicts get serialized automatically — the second group runs after the first merges. You don't have to do anything.

**What happens to worktrees if an agent fails?**
They're still deleted. The failed branch is deleted too. The sprint log records what failed and why. No disk waste either way.

**Does this work with any language/framework?**
Yes. The worktree isolation is pure git — it works with any codebase. The conflict analysis uses keyword matching on your task descriptions to detect likely file overlaps.

**How many agents can run at once?**
Up to 5 is recommended. Beyond that the coordinator review overhead starts to outweigh the parallelism benefit.

**Does this cost more tokens?**
The total token cost is similar to running the tasks sequentially — you're just running them at the same time. The saving is in wall-clock time. To reduce cost, assign `reviewer` and `tester` tasks to cheaper models like Haiku.
