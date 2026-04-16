---
name: claude-code-longrun
description: Use for Claude Code tasks that are long-running, iterative, or need persistent context across multiple rounds. Trigger when the user asks to use Claude Code for a complex coding task, wants tmux-based persistence, wants to reuse Claude Code context between follow-ups, or when a one-shot `claude --print` flow would likely lose too much context. Prefer this over generic coding-agent guidance for Claude Code sessions expected to run for many minutes or across multiple interactions.
---

# Claude Code Longrun

Use this skill when **Claude Code** is the requested tool and the task is too large, too iterative, or too context-heavy for a one-shot `claude --print` call.

## Use this skill when

- The user explicitly wants **Claude Code**.
- The task is expected to run for many minutes.
- You will likely send follow-up instructions into the **same Claude Code session**.
- The prompt is long enough that putting it directly on the command line is awkward.
- The job benefits from persistent working context, scratch notes, or multiple review/fix rounds.

## Do not use this skill when

- The task is tiny and can finish in one `claude --print` call.
- The user asked for Codex, Pi, ACP harness, or another tool.
- The work must happen in a Discord thread via ACP harness. In that case use `sessions_spawn` with `runtime:"acp"`.

---

# TWO-AGENT ARCHITECTURE

| Agent | Role |
|-------|------|
| **Parent agent** (you) | Pre-create tmux session, write task file, spawn operator sub-agent |
| **Operator sub-agent** | Send task to Claude Code via tmux, monitor progress, collect results |

**CRITICAL**: The operator sub-agent does NOT create tmux sessions. The parent creates the tmux session BEFORE spawning the sub-agent.

---

# Phase 1: Parent Agent Workflow

### Step 1: Create tmux session

```bash
SESSION_NAME="claude-$(date +%s)"
tmux new-session -d -s "$SESSION_NAME"
```

Save `$SESSION_NAME` — you'll pass it to the sub-agent.

### Step 2: Write the task file

```javascript
write({
  path: "/root/.openclaw/workspace/scripts/task-[shortname].md",
  content: `## Goal
[One sentence describing the end result]

## Project Context
- repo or directory: [path]
- relevant files: [list]

## Constraints
- [any limitations]

## What to do
1. [concrete steps]

## Deliverable
[Summarize what to report back]`
});
```

### Step 3: Spawn the operator sub-agent

```javascript
sessions_spawn({
  runtime: "subagent",
  mode: "run",
  task: `You are the OPERATOR for a Claude Code longrun task.

CRITICAL: 
- Do NOT create tmux sessions. One already exists.
- Do NOT spawn more sub-agents.
- Do NOT analyze the code yourself — you must use Claude Code.

Tmux session name: ${SESSION_NAME}
Project directory: /root/workspace/my_note

1. Send this command to launch Claude Code:
   tmux send-keys -t ${SESSION_NAME} "cd /root/workspace/my_note && claude --permission-mode bypassPermissions" Enter
   sleep 4

2. Send the task:
   tmux send-keys -t ${SESSION_NAME} -l -- "$(cat /root/.openclaw/workspace/scripts/task-[shortname].md)"
   tmux send-keys -t ${SESSION_NAME} Enter

3. Wait 60 seconds, then check:
   tmux capture-pane -t ${SESSION_NAME} -p | tail -60

4. If Claude is still working, wait another 60s and check again. Do this up to 3 times.

5. When Claude signals done (look for completion markers or "Human:" prompt), collect final output:
   tmux capture-pane -t ${SESSION_NAME} -p | tail -150

6. Kill the session:
   tmux kill-session -t ${SESSION_NAME}

7. Report:
   - 任务状态: ✅ 完成 / ⚠️ 阻塞 / 🚨 失败
   - UI 风格总结
   - 主要问题
   - 优化建议`
});
```

### Step 4: Yield

```javascript
sessions_yield({ message: "Claude Code 已在 tmux 中启动，完成后会自动通知你。" });
```

---

# Phase 2: Operator Sub-Agent Workflow (READ ONLY — don't respawn)

You are the **operator**. Your job is to send commands to the EXISTING tmux session and monitor Claude Code.

**You did NOT create this session. It was created by the parent agent.**

Commands you MUST run (in order):

```bash
# 1. Launch Claude Code in the existing session
tmux send-keys -t ${SESSION_NAME} "cd /root/workspace/my_note && claude --permission-mode bypassPermissions" Enter
sleep 4

# 2. Send the task (the text of the task file)
tmux send-keys -t ${SESSION_NAME} -l -- "$(cat /root/.openclaw/workspace/scripts/task-[shortname].md)"
tmux send-keys -t ${SESSION_NAME} Enter

# 3. Wait and monitor (check every 60s, up to 3 times)
sleep 60
tmux capture-pane -t ${SESSION_NAME} -p | tail -60

# 4. If still running, wait more
sleep 60
tmux capture-pane -t ${SESSION_NAME} -p | tail -60

# 5. If still running, wait more  
sleep 60
tmux capture-pane -t ${SESSION_NAME} -p | tail -60

# 6. Collect final output
tmux capture-pane -t ${SESSION_NAME} -p | tail -150

# 7. Cleanup
tmux kill-session -t ${SESSION_NAME}
```

---

# tmux Cheat Sheet

| Command | Purpose |
|---------|---------|
| `tmux new-session -d -s <name>` | Create detached session |
| `tmux send-keys -t <name> -l -- "text"` | Send text to session |
| `tmux send-keys -t <name> Enter` | Press Enter |
| `tmux capture-pane -t <name> -p` | Read session output |
| `tmux capture-pane -t <name> -p \| tail -40` | Last 40 lines |
| `tmux kill-session -t <name>` | Stop session |
| `tmux list-sessions` | Show all sessions |

---

# Why tmux?

`claude --print` loses context between calls. tmux keeps Claude Code alive so:
- Context persists across "continue" requests
- Claude can run long investigations without timeout
- You can send follow-up commands into the same session

---

# Common Pitfalls

1. **Creating tmux session in sub-agent** — parent creates it, sub-agent uses it
2. **Polling too often** — If Claude is working, let it work. Check every 60s.
3. **Huge inline prompts** — Always use a task file instead.
4. **Forgetting bypassPermissions** — Use `--permission-mode bypassPermissions` for exec/write operations.

# References

- Task file template: `references/task-file-template.md`
- Operation examples: `references/operation-examples.md`
