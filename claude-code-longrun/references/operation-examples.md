# Claude Code Longrun Operation Examples

## Example: Complete Parent + Sub-Agent Workflow

### Parent Agent Side

```javascript
// 1. Create tmux session FIRST (before spawning sub-agent)
const SESSION_NAME = "claude-" + Date.now();
exec({ command: `tmux new-session -d -s ${SESSION_NAME}` });

// 2. Write task file
write({
  path: "/root/.openclaw/workspace/scripts/task-ui-analysis.md",
  content: `## Goal
分析笔记系统的 UI 风格，给出优化建议。

## Project Context
- repo: /root/workspace/my_note
- files: public/views/index.html, public/css/style.css, public/js/app.js

## What to do
1. 分析视觉风格和配色方案
2. 分析布局设计
3. 列出用户体验问题
4. 给出具体优化建议

## Deliverable
汇报：UI 风格总结 + 主要问题 + 优化建议`
});

// 3. Spawn sub-agent with the SESSION_NAME
sessions_spawn({
  runtime: "subagent",
  mode: "run",
  task: `You are the OPERATOR for a Claude Code longrun task.

CRITICAL: 
- Do NOT create tmux sessions. One already exists.
- Do NOT spawn more sub-agents.
- Do NOT analyze code yourself.

Tmux session name: ${SESSION_NAME}

1. Launch Claude Code:
tmux send-keys -t ${SESSION_NAME} "cd /root/workspace/my_note && claude --permission-mode bypassPermissions" Enter
sleep 4

2. Send the task:
tmux send-keys -t ${SESSION_NAME} -l -- "$(cat /root/.openclaw/workspace/scripts/task-ui-analysis.md)"
tmux send-keys -t ${SESSION_NAME} Enter

3. Monitor (every 60s, up to 3 times):
tmux capture-pane -t ${SESSION_NAME} -p | tail -60
sleep 60

4. Collect final output:
tmux capture-pane -t ${SESSION_NAME} -p | tail -150

5. Cleanup:
tmux kill-session -t ${SESSION_NAME}

6. Report results to parent.`
});

// 4. Yield
sessions_yield({ message: "Claude Code 已启动，完成后会自动通知你。" });
```

### Operator Sub-Agent Side (spawned)

The sub-agent receives the SESSION_NAME and executes the tmux commands listed above. It does NOT create the session, does NOT spawn further agents, and does NOT analyze code directly.

---

## tmux Session Lifecycle

```
Parent creates session
       ↓
Parent spawns sub-agent (passes SESSION_NAME)
       ↓
Sub-agent sends "claude --print" command to session
       ↓
Claude Code runs inside tmux (persists across turns)
       ↓
Sub-agent polls tmux capture-pane periodically
       ↓
Claude finishes → Sub-agent collects output
       ↓
Sub-agent kills session → Reports to parent
```

---

## Example: Continue Same Session (Follow-up)

When user says "继续" or "再改一下":

```bash
# Find the session name from /root/.openclaw/workspace/scripts/
ls /root/.openclaw/workspace/scripts/ | grep task-

# Send follow-up command to existing session
tmux send-keys -t <session-name> -l -- "继续修复 UI 问题，重点处理手绘风格统一和浮动动画"
tmux send-keys -t <session-name> Enter
```

## Example: Stop a Bad Run

```bash
tmux send-keys -t <session-name> C-c
tmux kill-session -t <session-name>
```

## Example: Tiny task (no tmux needed)

```bash
cd /root/workspace/my_note && claude --print '列出 public/css/style.css 中的所有颜色变量'
```
