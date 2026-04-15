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
- The work is a simple local edit you can do directly.
- The work must happen in a Discord thread via ACP harness. In that case use `sessions_spawn` with `runtime:"acp"`.

## Core rule

For **Claude Code long tasks**, prefer:

1. **tmux session** for persistence
2. **task file** for long instructions
3. **low-frequency monitoring** instead of frequent interruption

Avoid treating Claude Code like a fire-and-forget one-shot when the user actually wants a continuing working session.

## Why

`claude --print` is fine for short tasks, but it is not the best default when:

- you want to preserve context across multiple turns,
- the prompt is long,
- the task may branch into investigation, edits, tests, and fixes,
- or the user will probably say “继续”, “再改一下”, “顺手把这个也做了”.

The failure mode is usually **over-interrupting too early**, not Claude Code being unable to handle the work.

## Preferred workflow

### 1. Start a dedicated tmux session

Use a clear session name related to the task.

Example:

```bash
tmux new-session -d -s claude-auth-fix
```

If the project directory matters, start Claude from the correct working directory.

### 2. Write a task file

Do not stuff a long brief directly into shell quoting if you can avoid it.

Create a task file inside the target project or a temp path, for example:

- `./.openclaw-tasks/claude-task.md`
- `/tmp/claude-task-<slug>.md`

The task file should include:

- the user goal
- scope boundaries
- constraints
- repo/path context
- required checks or tests
- expected final output format

Keep task files free of unnecessary secrets. Prefer describing required credentials or environment assumptions instead of pasting tokens, private keys, or personal data.

See `references/task-file-template.md` for a reusable structure.

### 3. Launch Claude Code inside tmux

Preferred pattern:

```bash
cd /path/to/project && claude --permission-mode bypassPermissions
```

Then send Claude a short instruction telling it to read the task file and execute from there.

Example instruction sent into tmux:

```text
Read ./.openclaw-tasks/claude-task.md, follow it exactly, keep notes concise, and tell me when you are blocked or fully done.
```

## Monitoring strategy

### Default

Check output with `tmux capture-pane` at **low frequency**.

Good default cadence:

- first check after a short settling period
- then about every **5 minutes** for long tasks
- faster only when the agent is clearly waiting for input

### Do

- capture recent output
- look for explicit questions, blockers, test failures, or completion
- let Claude continue if it is clearly making progress

### Do not

- interrupt just because output paused briefly
- resend the task repeatedly
- kill and restart a healthy run too early

## Interaction pattern

Use tmux as the persistent shell, not as a chat transcript viewer.

Typical loop:

1. create tmux session
2. launch Claude Code in project dir
3. write task file
4. send a short “read this task file” instruction
5. capture output occasionally
6. only intervene if Claude asks for input, stalls clearly, or finishes

## Recommended command patterns

### Start session

```bash
tmux new-session -d -s <session-name>
```

### Start Claude inside that session

```bash
tmux send-keys -t <session-name> 'cd /path/to/project && claude --permission-mode bypassPermissions' Enter
```

### Send the task instruction safely

```bash
tmux send-keys -t <session-name> -l -- 'Read ./.openclaw-tasks/claude-task.md and execute it. If blocked, say exactly what you need.'
sleep 0.1
tmux send-keys -t <session-name> Enter
```

### Check recent output

```bash
tmux capture-pane -t <session-name> -p | tail -40
```

### Check entire scrollback when needed

```bash
tmux capture-pane -t <session-name> -p -S -
```

## Decision guide

### Use one-shot `claude --print` when

- the ask is narrow
- no persistent context is needed
- a single response is likely enough

### Use this skill when

- the ask is open-ended or investigative
- the user is likely to iterate on Claude's work
- you expect code, test, revise, and re-run cycles
- preserving Claude's local context is part of the value

## Output expectations

When using this skill, keep the human updated only when something meaningful changes:

- started and where it is running
- blocked and what input is needed
- milestone reached
- finished with concrete results

Do not spam the user with minor polling updates.

## Common pitfalls

- Starting Claude with a huge inline prompt instead of a task file
- Polling too often and mistaking quiet work for failure
- Forgetting to run from the correct project directory
- Using this pattern for tiny tasks that should have been one-shot
- Embedding local machine assumptions or private workflow details that make the skill less portable

## References

- Task file template: `references/task-file-template.md`
- Operation examples: `references/operation-examples.md`
