# Ralph Eats Beads 🦀

Autonomous AI coding loop using [bd (beads)](https://github.com/rawwerks/bd) for task management with parallel tmux subagents.

Ships features while you sleep.

## Installation

### Claude Code (Interactive)

```
/plugin marketplace add rawwerks/ralph-eats-beads
/plugin install ralph-eats-beads@ralph-eats-beads
```

### Claude CLI

```bash
claude plugin marketplace add rawwerks/ralph-eats-beads
claude plugin install ralph-eats-beads@ralph-eats-beads
```

## What It Does

Ralph picks issues from `bd ready`, spawns parallel Claude agents in tmux windows to implement them, and closes issues on success. Each iteration:

1. Parent agent checks `bd ready` for available work
2. Spawns 3-5 subagent tmux windows (one per issue)
3. Each subagent implements ONE issue, commits, closes it
4. Watchdog monitors for stuck agents and nudges/kills them
5. Loop repeats until no work remains or max iterations hit

```
ralph (tmux session)
├── parent (window 0) - orchestrator
├── watchdog (window 1) - health monitor  
├── issue-1 (window 2) - subagent
├── issue-2 (window 3) - subagent
└── issue-3 (window 4) - subagent
```

## Prerequisites

- **[bd](https://github.com/rawwerks/bd)** - Issue tracker (beads)
- **tmux** - Terminal multiplexer
- **claude** - Claude CLI with `--dangerously-skip-permissions`

## Quick Start

```bash
# 1. Create issues to work on
bd create --type epic "Feature: User Auth"
bd create "Add login form" --parent <epic-id>
bd create "Add validation" --parent <epic-id>

# 2. Start Ralph (default: 10 iterations max)
./scripts/ralph.sh

# 3. Or specify max iterations
./scripts/ralph.sh 25

# 4. Watch progress
tmux attach -t ralph-<project>
```

## How Iterations Work

- **Max iterations** = how many times the parent can spawn a batch of subagents
- Each iteration handles multiple issues in parallel (3-5 concurrent)
- Loop exits early if `bd ready` returns no work
- Default timeout: 1 hour per iteration

## On Failure

- Subagent adds comment: `bd comments add <id> "Blocked: <reason>"`
- Issue stays open for next iteration or manual fix
- Watchdog kills stuck windows after 2 nudge attempts

## Files

| File | Purpose |
|------|---------|
| `scripts/ralph.sh` | Main entry point |
| `scripts/planner.sh` | Generate issues from requirements |
| `scripts/watchdog.sh` | Health monitor |
| `SKILL.md` | Full agent instructions |

## License

MIT
