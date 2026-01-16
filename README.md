# Ralph Eats Beads 🦀

Autonomous AI coding loop using bd (beads) for task management with parallel tmux subagents.

Ships features while you sleep.

## Claude Code Plugin Installation

Add this skill to Claude Code:

```bash
# Via Claude Code CLI
claude skill add rawwerks/ralph-eats-beads
```

Or add to your `.claude/settings.json`:

```json
{
  "skills": [
    "rawwerks/ralph-eats-beads"
  ]
}
```

## Overview

Ralph runs Claude agents in a loop, picking tasks from `bd ready`, implementing them, and closing them on success. Uses tmux for observability and supports parallel subagents.

```
ralph (tmux session)
├── parent (window 0) - orchestrator Claude
├── watchdog (window 1) - health monitor Claude
├── BD-001 (window 2) - subagent
├── BD-002 (window 3) - subagent
└── BD-003 (window 4) - subagent
```

## Prerequisites

- **bd** (beads) initialized with issues/epic
- **tmux** installed
- **Claude CLI** (`claude` command available)

## Quick Start

```bash
# Create your epic and issues
bd create --type epic "Feature: User Authentication"
bd create "Add login form" --parent BD-001 --priority 1

# Start Ralph (25 max iterations)
scripts/ralph.sh 25

# Attach to watch
tmux attach -t ralph
```

## Full Documentation

See [SKILL.md](SKILL.md) for complete documentation including:
- Planner usage (AI-generated issue plans)
- Watchdog configuration
- Observer commands
- Best practices

## License

MIT
