# tmux Commands Reference

Concrete commands the Supervisor uses to manage the Coding CLI's tmux session.

## Launch

```bash
# Create a tmux session for the task
tmux new-session -d -s SESSION_NAME -c /path/to/repo

# Start the Coding CLI inside the session
# Use the strongest available model. Examples:
tmux send-keys -t SESSION_NAME 'codex --model gpt-5.4 --full-auto' Enter
tmux send-keys -t SESSION_NAME 'claude --model opus --dangerously-skip-permissions' Enter
tmux send-keys -t SESSION_NAME 'gemini --model auto' Enter
```

After the Coding CLI starts, send the task prompt:

```bash
# Short prompt — send directly
tmux send-keys -t SESSION_NAME 'Your prompt here' Enter

# Multi-line prompt — write to file, load into tmux buffer, then paste
cat <<'EOF' > /tmp/task-prompt.txt
Implement the auth middleware.

Constraints:
- Use JWT with RS256
- Token refresh on expiry

Work autonomously. Summarize progress every 10 minutes.
Run tests before claiming completion.
EOF

tmux load-buffer /tmp/task-prompt.txt
tmux paste-buffer -t SESSION_NAME
tmux send-keys -t SESSION_NAME Enter
```

Record: session name, launch time, chosen CLI and model.

## Monitor

```bash
# Capture last 100 lines of pane output
tmux capture-pane -t SESSION_NAME -p -S -100

# Check if session exists
tmux has-session -t SESSION_NAME 2>/dev/null && echo "alive" || echo "dead"

# Check what process is running inside the session
tmux list-panes -t SESSION_NAME -F '#{pane_current_command}'
```

### Reading the signals

| Signal | Meaning |
|--------|---------|
| Editing files, searching code | Active implementation |
| Running tests, linters, builds | Validation phase |
| Waiting for input, repeating same error | Blocked / stalled |
| Clean prompt return, "done" summary | Finished — start review |
| Shell prompt with no CLI running | Crashed or exited — recover |

## Recovery

### Detect state

```bash
# Session gone?
tmux has-session -t SESSION_NAME 2>/dev/null || echo "SESSION GONE"

# CLI exited? (pane shows shell instead of CLI)
tmux list-panes -t SESSION_NAME -F '#{pane_current_command}'
# If output is "zsh" or "bash" → CLI exited

# Capture last output for diagnosis
tmux capture-pane -t SESSION_NAME -p -S -200
```

### Restart the Coding CLI

```bash
# If session exists but CLI died, restart inside it
tmux send-keys -t SESSION_NAME 'codex --model gpt-5.4 --full-auto' Enter

# If session is gone, recreate it
tmux new-session -d -s SESSION_NAME -c /path/to/repo
tmux send-keys -t SESSION_NAME 'codex --model gpt-5.4 --full-auto' Enter
```

### Resume prompt

After restarting, send a resume prompt so the Coding CLI picks up where it left off:

```
Resume the task.
Goal: <original goal>.
Current repo state: <summary from git status/diff>.
Completed so far: <what was done before crash>.
Last blocker: <error or stall reason>.
Continue from here. Avoid redoing finished work. Run validation before claiming completion.
```

## Send review feedback

When the Supervisor finds issues, send concrete feedback directly into the Coding CLI's input:

```bash
tmux send-keys -t SESSION_NAME 'Review feedback: the new cache invalidation path updates Redis but leaves in-memory state stale in the worker. Please fix the consistency bug, add a test proving both layers stay in sync, and summarize the root cause.' Enter
```

Good feedback follows this pattern:
1. State the specific issue
2. Point to files or behaviors
3. Ask for a fix and verification

Then monitor the pane to see the Coding CLI's response. It may fix the issue, or it may push back with a valid reason — read its output and decide whether to accept or send further feedback.

## Review checklist

When the Coding CLI claims completion, verify:

- [ ] `git diff` — changes match the goal
- [ ] Tests ran and passed
- [ ] Acceptance criteria actually met (not just claimed)
- [ ] No unsafe migrations or destructive commands
- [ ] No unnecessary complexity or scope creep
- [ ] Error handling adequate

If anything fails, send feedback into tmux and iterate until consensus.
