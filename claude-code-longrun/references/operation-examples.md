# Claude Code Longrun Operation Examples

These are practical patterns for running Claude Code as a persistent tmux-backed worker.

## Example 1: Start a long task from scratch

Use when the user asks Claude Code to handle a large coding task in a real repo.

```bash
PROJECT=~/projects/my-repo
SESSION=claude-my-repo-auth
TASK_DIR="$PROJECT/.openclaw-tasks"
TASK_FILE="$TASK_DIR/auth-fix.md"

mkdir -p "$TASK_DIR"
cat > "$TASK_FILE" <<'EOF'
## Goal
Fix the auth refresh flow bug without changing unrelated login behavior.

## Project Context
- repo: ~/projects/my-repo
- inspect auth middleware, refresh token handling, and API client retry path first

## Constraints
- keep changes minimal
- do not change public API contracts
- run targeted tests if available

## What to do
1. Trace the current refresh flow.
2. Find the actual cause of the bug.
3. Implement the smallest safe fix.
4. Run relevant validation.
5. Summarize files changed, commands run, and remaining risks.
EOF

tmux new-session -d -s "$SESSION"
tmux send-keys -t "$SESSION" "cd '$PROJECT' && claude --permission-mode bypassPermissions" Enter
sleep 2

tmux send-keys -t "$SESSION" -l -- "Read ./.openclaw-tasks/auth-fix.md and execute it. If blocked, stop and ask one clear question."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

## Example 2: Poll without over-interrupting

Use when the task is already running and you only want to check progress.

```bash
SESSION=claude-my-repo-auth

tmux capture-pane -t "$SESSION" -p | tail -40
```

If Claude is actively investigating, editing, or testing, leave it alone.

A good default is to check roughly every 5 minutes for longer tasks.

## Example 3: Claude is blocked and needs input

If output shows a question or a request for a decision, send one compact instruction back.

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" -l -- "Use the existing retry contract. Do not introduce a new config flag. Continue with the minimal fix."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

## Example 4: Continue the same session in a follow-up turn

Use when the user says something like “继续”, “顺手把测试也补了”, or “再把日志整理一下”.

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" -l -- "Continue in the same branch. Add or update targeted tests for the refresh flow, then summarize what changed."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

This is the main reason to prefer tmux over a one-shot `claude --print` flow.

## Example 5: Inspect full scrollback when recent output is not enough

```bash
SESSION=claude-my-repo-auth

tmux capture-pane -t "$SESSION" -p -S -
```

Use this sparingly. Usually the last 30 to 60 lines are enough.

## Example 6: Stop a clearly bad run

Only do this when Claude is truly stuck, looping, or working on the wrong thing.

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" C-c
```

Then either:

- send a corrected instruction into the same session, or
- start a fresh session if the current context is too polluted

## Example 7: Tiny task, do not use tmux

For a narrow one-shot ask, use `claude --print` instead.

```bash
cd ~/projects/my-repo && claude --permission-mode bypassPermissions --print 'Explain where the refresh token is validated and list the relevant files.'
```

Use the longrun skill only when persistence and follow-up context actually matter.
