---
name: claude-code-longrun
description: Use for Claude Code tasks that are long-running, iterative, or need persistent context across multiple rounds. Trigger when the user asks to use Claude Code for a complex coding task, wants tmux-based persistence, wants to reuse Claude Code context between follow-ups, or when a one-shot `claude --print` flow would would likely lose too much context. Prefer this over generic coding-agent guidance for Claude Code sessions expected to run for many minutes or across multiple interactions.
---

# Claude Code Longrun

Use this skill when **Claude Code** is the requested tool and the task is too large, too iterative, or too context-heavy for a one-shot `claude --print` call.

## Decision Guide

Not every coding task needs tmux. Here's when to use which:

| Situation | Approach |
|-----------|----------|
| Narrow, one-off ask | `claude --print` directly |
| Simple local edit you can do | Use native tools directly |
| Complex/long task, no follow-up | `claude --print` with task file |
| Complex task + likely follow-ups ("继续", "再改一下") | **This skill** — tmux-backed session |
| ACP harness thread request | `sessions_spawn` with `runtime:"acp"` instead |

**Use this skill when:**
- The user explicitly wants **Claude Code**
- The task is expected to run for many minutes
- You will likely send follow-up instructions into the **same Claude Code session**
- The job benefits from persistent working context across review/fix cycles

**Do NOT use this skill when:**
- The task is tiny and can finish in one `claude --print` call
- The user asked for Codex, Pi, or ACP harness
- The work must happen in a Discord thread (use `sessions_spawn` with `runtime:"acp"`)

---

# TWO-AGENT ARCHITECTURE

| Agent | Role |
|-------|------|
| **Parent agent** (you) | Create tmux session, write task file, spawn operator sub-agent |
| **Operator sub-agent** | Send task to Claude Code via tmux, monitor, make workflow judgments |

**Design goals:**
- Parent creates infrastructure (tmux session) before spawning — avoids sub-agent exec failures
- Sub-agent does NOT spawn further sub-agents — prevents recursive spawning loops
- Sub-agent may make workflow judgments (is Claude blocked? should I intervene?) but must not independently complete the main task — Claude Code does the actual work
- tmux session is preserved by default — follow-up turns reuse the same session

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
  path: "/path/to/task-[shortname].md",
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

Tmux session: ${SESSION_NAME}
Project directory: [path/to/project]

1. Launch Claude Code:
tmux send-keys -t ${SESSION_NAME} "cd [PROJECT_DIR] && claude --permission-mode bypassPermissions" Enter
sleep 4

2. Tell Claude to read and execute the task file:
tmux send-keys -t ${SESSION_NAME} -l -- "Read /path/to/task-[shortname].md and execute it. If blocked, stop and ask one clear question."
tmux send-keys -t ${SESSION_NAME} Enter

3. Monitor — check tmux output every 2-5 minutes. Do NOT interrupt Claude if it is actively working.

4. When Claude signals done or is blocked, collect output:
tmux capture-pane -t ${SESSION_NAME} -p | tail -150

5. Report:
- 任务状态: ✅ 完成 / ⚠️ 阻塞 / 🚨 失败
- tmux session: ${SESSION_NAME} (preserve it — do NOT kill)
- summary: [what happened]`
});
```

### Step 4: Yield

```javascript
sessions_yield({ message: "Claude Code 已在 tmux 中启动，完成后会自动通知你。" });
```

---

# Phase 2: Operator Sub-Agent Workflow

**Role**: Operator — launch Claude Code, monitor, make workflow decisions. Claude Code does the actual work.

**Operator rules:**
- ✅ Send task instructions to Claude via tmux
- ✅ Make workflow judgments (is Claude blocked? has it gone off track?)
- ✅ Send corrective instructions when Claude is stuck
- ✅ Report results to parent
- ❌ Do NOT analyze the code or complete the main task yourself
- ❌ Do NOT create tmux sessions (parent creates them)
- ❌ Do NOT spawn more sub-agents

### Monitoring guidance

**Low-frequency monitoring is correct behavior.** For genuinely long-running tasks, over-monitoring is worse than under-monitoring.

Default cadence:
1. First check after Claude has had a few seconds to start (2-5s settling)
2. Then check every **2-5 minutes** — not every 60 seconds
3. Only intervene when Claude is clearly blocked, stuck, or asking a question

Signs Claude is working well:
- Output shows investigation, editing, running commands
- Progress is being made

Signs you should intervene:
- Claude is asking a clear question that needs a decision
- Claude has gone quiet for an unusually long time (>10 min on a complex task)
- Claude is clearly stuck in a loop

### Session preservation

**Do NOT kill the tmux session by default.** The session persists so follow-up turns ("继续", "再改一下", "add tests") can send new instructions into the same Claude Code context.

Only kill the session when:
- The task is fully complete and no follow-ups are expected
- Claude has been stuck and you need to start fresh
- The parent explicitly requests cleanup

---

# tmux Cheat Sheet

| Command | Purpose |
|---------|---------|
| `tmux new-session -d -s <name>` | Create detached session |
| `tmux send-keys -t <name> -l -- "text"` | Send text to session |
| `tmux send-keys -t <name> Enter` | Press Enter |
| `tmux capture-pane -t <name> -p` | Read session output |
| `tmux capture-pane -t <name> -p \| tail -40` | Last 40 lines |
| `tmux capture-pane -t <name> -p \| tail -150` | Last 150 lines (for final report) |
| `tmux kill-session -t <name>` | Stop session |
| `tmux list-sessions` | Show all sessions |

---

# Common Pitfalls

| Pitfall | Why it's bad | Prevention |
|---------|-------------|------------|
| Sub-agent creates tmux session | Exec may fail or behave unexpectedly in sub-agent context | Parent creates session before spawning |
| Recursive sub-agent spawning | Sub-agent spawns another sub-agent → infinite loop | Operator is explicitly final — no further spawning |
| Operator completes task itself | Undermines the whole point of using Claude Code | Operator sends to Claude, Claude does the work |
| Over-monitoring (polling every 60s) | Interrupts Claude's work, wastes resources | Check every 2-5 minutes |
| Default session teardown | Kills the one thing this skill is designed to preserve | Session cleanup is conditional, not default |
| Hard-coded environment paths | Makes skill non-portable | Examples use [path] / [PROJECT_DIR] placeholders |
| Pasting full task via tmux send-keys | Noisy and fragile for long prompts | Claude reads the task file directly |

---

# Why tmux?

`claude --print` loses context between calls. tmux keeps Claude Code alive so:
- Context persists across "continue" requests
- Claude can run long investigations without timeout
- You can send follow-up commands into the same session
- Multiple review/fix cycles keep their context

---

# References

- Task file template: `references/task-file-template.md`
- Operation examples: `references/operation-examples.md`
