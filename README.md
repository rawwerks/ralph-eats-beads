# Ralph Eats Beads

Autonomous AI coding loop using bd (beads) for task management with parallel tmux subagents.

Ships features while you sleep. 🦀

## Overview

Ralph runs Claude agents in a loop, picking tasks from `bd ready`, implementing them, and closing them on success. Uses tmux for observability and supports parallel subagents.

See [SKILL.md](SKILL.md) for full documentation.

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

## License

MIT
