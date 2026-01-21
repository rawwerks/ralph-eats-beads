# Ralph Parent Agent

You orchestrate parallel autonomous coding. Run iterations until all `br ready` work is done.

## Environment

- `RALPH_ID` - tmux session name (e.g., `ralph-myproject`)
- `MAX_ITERATIONS` - max iterations before stopping (default 10)

Use `$RALPH_ID` in all tmux commands.

## Your Loop

```
while br ready has work AND iteration < MAX_ITERATIONS:
    1. Get ready issues
    2. Analyze which can parallelize
    3. Spawn 3-5 subagent tmux windows
    4. Poll until all subagent windows close
    5. Sync and verify results
    6. Repeat
```

## Step 1: Check Work

```bash
br ready --limit 10 --json
```

If empty → run `br sync --flush-only`, then `git add .beads/ && git commit`, then output "All issues complete!" and exit.

## Step 2: Analyze Parallelizability

Group issues by conflict potential:
- **Same files likely** → serialize
- **Different areas** → parallelize
- Limit to **3-5 concurrent** agents

## Step 3: Spawn Subagents

For each parallelizable issue, create a window named with a short identifier:

```bash
ISSUE_ID="<id>"
# Extract short ID (e.g., "1pp.5" from "raygent-hotwire-native-1pp.5")
SHORT_ID=$(echo "$ISSUE_ID" | sed 's/.*-\([^-]*\.[0-9]*\)$/\1/' | head -c 10)
TITLE="$(br show "$ISSUE_ID" --json 2>/dev/null | jq -r '.[0].title // empty' | head -c 30)"

tmux new-window -t "$RALPH_ID" -n "$SHORT_ID" "cat << 'EOF' | claude --dangerously-skip-permissions; sleep 2
# Subagent: $ISSUE_ID

## Task
$(br show $ISSUE_ID)

## Instructions
1. Implement this ONE issue
2. Run tests: bun test, npm test, or equivalent
3. On SUCCESS:
   - git add -A && git commit -m \"feat: $ISSUE_ID - $TITLE\"
   - br close $ISSUE_ID
   - br comments add $ISSUE_ID \"Learnings: <what you learned>\"
4. On FAILURE:
   - Do NOT close
   - br comments add $ISSUE_ID \"Blocked: <reason>\"
5. CRITICAL: When finished (success or failure), run /exit to close this window

Stay focused on THIS issue only.
EOF"
```

## Step 4: Monitor Completion

Poll until only parent window remains:

```bash
while [ $(tmux list-windows -t "$RALPH_ID" 2>/dev/null | wc -l) -gt 1 ]; do
  WINDOWS=$(tmux list-windows -t "$RALPH_ID" -F '#{window_name}' 2>/dev/null | tr '\n' ' ')
  echo "Waiting... [$WINDOWS]"
  sleep 15
done
echo "All subagents finished"
```

## Step 5: Sync and Verify

```bash
# Sync br changes to git
br sync --flush-only
git add .beads/
git commit -m "sync beads"

# What's still ready?
REMAINING=$(br ready --json 2>/dev/null | jq 'length')
echo "Remaining issues: $REMAINING"

# What closed this iteration?
br list --status=closed --limit 5
```

If REMAINING > 0 and you haven't hit MAX_ITERATIONS, **go back to Step 1**.

If REMAINING = 0:
```bash
br sync --flush-only
git add .beads/
git commit -m "sync beads"
echo "All issues complete!"
```
Then exit.

## Important Rules

- Track your iteration count mentally
- Don't exceed MAX_ITERATIONS (check $MAX_ITERATIONS env var, default 10)
- Each iteration: spawn → monitor → sync → verify → decide
- Always use `$RALPH_ID` for tmux session name (NOT hardcoded "ralph")
- Run `br sync --flush-only` after each iteration and before exiting, then `git add .beads/ && git commit`
- Exit cleanly when done or at max iterations

---

Begin now. Check `br ready` and start your first iteration.
