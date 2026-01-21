---
name: ralph-eats-beads
description: Autonomous AI coding loop using br for task management with parallel tmux subagents. Ships features while you sleep.
license: MIT
---

# Ralph BR - Parallel Autonomous Coding Loop

**Contract**: `skills-lib-lkv` (Ralph Contract v1)

Ralph runs Claude agents in a loop, picking tasks from `br ready`, implementing them, and closing them on success. This version uses tmux for observability and supports parallel subagents.

## Architecture

```
ralph (tmux session)
├── parent (window 0) - orchestrator Claude
├── watchdog (window 1) - health monitor Claude (optional)
├── br-001 (window 2) - subagent
├── br-002 (window 3) - subagent
└── br-003 (window 4) - subagent
```

## Prerequisites

- **br** initialized with issues/epic
- **tmux** installed
- **Claude CLI** (`claude` command available)

## Quick Start

### Option A: Manual Plan (you create issues)

```bash
# 1. Create your epic and issues
br create --type epic "Feature: User Authentication"
br create "Add login form" --parent br-001 --priority 1
br create "Add email validation" --parent br-001 --priority 2
br create "Add auth server action" --parent br-001 --priority 3

# 2. Start Ralph (25 max iterations)
scripts/ralph.sh 25

# 3. Attach to watch
tmux attach -t ralph
```

### Option B: AI-Generated Plan (planner creates issues)

```bash
# 1. Let Claude create the plan from requirements
scripts/planner.sh requirements.md

# Or inline:
scripts/planner.sh <<< "Add OAuth2 authentication with Google and GitHub providers"

# Or pipe from anywhere:
cat feature-spec.md | scripts/planner.sh

# 2. Watch the planner
tmux attach -t ralph-plan

# 3. Once plan is created, implement it
scripts/ralph.sh 25
```

## Planning vs Implementation

| Script | Session | Purpose |
|--------|---------|---------|
| `planner.sh` | ralph-plan | **Recursive loop** - creates/refines br issues until plan is complete |
| `ralph.sh` | ralph | **Parallel loop** - implements issues from `br ready` |

**Workflow:**
```
Requirements → planner.sh (iterates) → br issues → ralph.sh (parallel) → Implemented code
```

Both loops support `MAX_ITERATIONS` - planner defaults to 3, implementation defaults to 10.

## How It Works

**Key principle**: Bash manages the iteration loop, each iteration gets fresh Claude context.

```
ralph.sh (bash loop)
├── Iteration 1
│   ├── Spawn fresh Claude parent
│   ├── Parent spawns subagents, then exits
│   ├── Bash waits for subagent windows to close
│   ├── br sync --flush-only
│   ├── git add .beads/
│   ├── git commit -m "sync beads"
│   └── Check br ready → continue or done
├── Iteration 2
│   └── (repeat with fresh context)
└── ...
```

1. Bash checks `br ready` for available work
2. Bash spawns fresh Claude parent for this iteration
3. Parent runs `br info`, analyzes parallelizability
4. Parent spawns subagent tmux windows (3-5 concurrent), then exits
5. Bash waits for all subagent windows to close
6. Each subagent implements ONE issue, commits, closes issue
7. Bash syncs br and loops until no work remains

## Observer Commands

```bash
# Attach to see everything
tmux attach -t ralph

# Tile all panes
tmux select-layout -t ralph tiled

# Watch specific agent
tmux select-window -t ralph:br-123

# Capture agent output
tmux capture-pane -t ralph:br-123 -p -S -1000 > agent.log

# Kill stuck agent
tmux kill-window -t ralph:br-123

# Kill entire session
tmux kill-session -t ralph
```

## Planner Details

The planner (`scripts/planner.sh`) runs a **recursive loop** that iterates until the br plan fully covers the input requirements:

```
while iteration < MAX_ITERATIONS:
    1. Read input plan file
    2. Check current br state (existing issues)
    3. Compare: what's in the plan but missing from br?
    4. If nothing missing → exit with summary
    5. Create missing issues with dependencies
    6. Repeat
```

Each iteration the planner:
1. **Re-reads the plan** - Keeps the source of truth in focus
2. **Audits br state** - What stories exist? What's missing?
3. **Fills gaps** - Creates issues for uncovered sections
4. **Sets dependencies** - Ensures correct build order

This ensures complex plans with many sections get fully decomposed into implementable stories.

### Input Formats

```bash
# From file
scripts/planner.sh requirements.md
scripts/planner.sh feature-spec.txt

# Inline (heredoc)
scripts/planner.sh << 'EOF'
Add user authentication:
- OAuth2 with Google and GitHub
- Session management with JWT
- Protected routes middleware
EOF

# One-liner
scripts/planner.sh <<< "Add dark mode with system preference detection"

# Pipe from anywhere
curl -s https://example.com/spec.md | scripts/planner.sh
gh issue view 123 --json body -q .body | scripts/planner.sh
```

### What the Planner Creates

```
Epic: AUTH-001 "Feature: OAuth2 Authentication"
├── AUTH-002 "Add OAuth2 configuration" [ready]
├── AUTH-003 "Create Google provider" [depends: AUTH-002]
├── AUTH-004 "Create GitHub provider" [depends: AUTH-002]
├── AUTH-005 "Add callback handlers" [depends: AUTH-003, AUTH-004]
└── AUTH-006 "Add protected route middleware" [depends: AUTH-005]
```

Stories include:
- Clear acceptance criteria
- File paths to modify
- References to existing patterns
- Testing requirements

## Watchdog (Active Health Monitor)

The watchdog is a separate Claude agent that monitors Ralph sessions and unsticks them automatically. Unlike the passive monitor, it takes action:

```bash
# Start watchdog for a ralph session (30 second check interval)
scripts/watchdog.sh ralph-myproject 30

# It runs in its own tmux window within the session
tmux select-window -t ralph-myproject:watchdog
```

The watchdog:
- **Detects idle agents** - Looks for Claude CLI prompts indicating waiting state
- **Checks issue status** - Queries br to see if work is complete
- **Nudges stuck agents** - Sends tmux keystrokes to prompt them to continue
- **Kills completed windows** - Cleans up windows where the issue is closed
- **Kicks the parent** - Sends Enter if parent is idle with work remaining

This keeps Ralph flowing autonomously without human intervention.

## Passive Monitor (Logging Only)

For unattended runs where you just want logging without intervention:

```bash
# Start monitor (checks every 20 min, logs to /tmp/ralph-monitor.log)
nohup scripts/monitor.sh 20 &

# Check status
tail -f /tmp/ralph-monitor.log
```

The passive monitor detects:
- **STOPPED**: Ralph session died unexpectedly
- **STUCK**: Only parent window but work remains (2 consecutive checks)
- **COMPLETE**: No ready work and only parent window
- **RUNNING**: Normal operation with active subagents

## br (beads_rust) Workflow

**Note:** `br` is non-invasive and never executes git commands. After `br sync --flush-only`, you must manually run `git add .beads/ && git commit`.

### Setup Work

```bash
# Create epic
br create --type epic "Feature: Dark Mode"  # → br-001

# Create stories
br create "Add theme toggle" --parent br-001 --priority 1
br create "Add theme context" --parent br-001 --priority 2
br create "Update components" --parent br-001 --priority 3

# Add dependencies (br ready respects these)
br dep add br-004 br-003  # br-004 depends on br-003
```

### Monitor Progress

```bash
br ready              # What's workable?
br epic status br-001 # Epic completion
br blocked            # What's stuck?
br history            # Recent changes
```

### After Completion

```bash
br list --state closed --parent br-001  # All closed?
br show br-002 --comments               # Review learnings
br epic close-eligible                  # Close the epic
```

## Best Practices

### Right-Sized Stories

```
BAD:  "Build authentication system"
GOOD: "Add login form with email/password fields"
      "Add client-side email validation"
      "Create auth server action"
```

### Clear Acceptance Criteria

In issue body:
```
Acceptance Criteria:
- [ ] Email field with validation
- [ ] Password field with show/hide
- [ ] typecheck passes
- [ ] tests pass
```

### Learnings Accumulate

Agents add learnings via `br comments`. Future iterations benefit:
```bash
br show br-005 --comments
br search "migration pattern"
```

## Not Suitable For

- Exploratory work without clear criteria
- Major refactors spanning entire codebase
- Security-critical code
- Anything needing human review before commit
