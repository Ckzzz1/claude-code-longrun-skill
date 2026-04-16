# Claude Code Longrun Operation Examples

## Example: Complete Parent + Operator Workflow

### Parent Agent Side

```javascript
// 1. Create tmux session FIRST (before spawning sub-agent)
const SESSION_NAME = "claude-" + Date.now();
exec({ command: `tmux new-session -d -s ${SESSION_NAME}` });

// 2. Write task file
write({
  path: "/path/to/task-analysis.md",
  content: `## Goal
Analyze the UI style of the note-taking app and give improvement suggestions.

## Project Context
- repo or directory: /path/to/project
- files: public/views/index.html, public/css/style.css, public/js/app.js

## What to do
1. Analyze visual style and color scheme
2. Analyze layout design
3. List UX problems
4. Give specific improvement suggestions

## Deliverable
Report: UI style summary + main issues + improvement suggestions`
});

// 3. Spawn sub-agent — pass SESSION_NAME
sessions_spawn({
  runtime: "subagent",
  mode: "run",
  task: `You are the OPERATOR for a Claude Code longrun task.

Tmux session: ${SESSION_NAME}

1. Launch Claude Code:
tmux send-keys -t ${SESSION_NAME} "cd /path/to/project && claude --permission-mode bypassPermissions" Enter
sleep 4

2. Tell Claude to read and execute the task file:
tmux send-keys -t ${SESSION_NAME} -l -- "Read /path/to/task-analysis.md and execute it. If blocked, stop and ask one clear question."
tmux send-keys -t ${SESSION_NAME} Enter

3. Monitor — check tmux output every 2-5 minutes. Do NOT interrupt Claude if it is working.

4. When Claude signals done or is blocked:
tmux capture-pane -t ${SESSION_NAME} -p | tail -150

5. Report to parent with:
- 任务状态: ✅ 完成 / ⚠️ 阻塞 / 🚨 失败
- tmux session: ${SESSION_NAME} (do NOT kill — preserve for follow-ups)
- summary: [what happened]`
});

// 4. Yield
sessions_yield({ message: "Claude Code 已在 tmux 中启动，完成后会自动通知你。" });
```

### Operator Sub-Agent Side (spawned)

Receives SESSION_NAME, then:
1. Sends "claude --print" command to the existing tmux session
2. Tells Claude to read the task file (NOT paste full contents)
3. Monitors at low frequency (every 2-5 minutes)
4. Makes workflow judgments — may intervene if Claude is stuck
5. Reports results to parent, preserving the tmux session

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
Sub-agent monitors at low frequency (2-5 min intervals)
       ↓
Claude finishes OR signals blocked → Sub-agent collects output
       ↓
tmux session PRESERVED for follow-up turns (NOT killed by default)
       ↓
Follow-up turns send new instructions into same session
```

---

## Example: Follow-up Turn (Continue Same Session)

When user says "继续", "再改一下", or "also add tests":

```bash
# Find the tmux session
tmux list-sessions

# Send follow-up instruction to the existing session
tmux send-keys -t <session-name> -l -- "Continue fixing the UI issues. Add specific CSS improvements for the sidebar float animation."
tmux send-keys -t <session-name> Enter

# Monitor
tmux capture-pane -t <session-name> -p | tail -60
```

The key benefit of tmux: Claude preserves context from the original task.

---

## Example: Claude is Blocked — Send Intervention

If tmux output shows Claude is waiting for a decision:

```bash
tmux send-keys -t <session-name> -l -- "Use the existing color palette. Do not introduce new dependencies. Continue with the minimal fix."
tmux send-keys -t <session-name> Enter
```

---

## Example: Stop a Bad Run

```bash
# Interrupt Claude
tmux send-keys -t <session-name> C-c

# Send corrected instruction OR kill session
tmux kill-session -t <session-name>
```

---

## Example: Task is Small — Use One-Shot Instead

For tiny asks, do NOT use this skill:

```bash
cd /path/to/project && claude --print 'List the color variables in public/css/style.css'
```

Use longrun only when persistence and follow-up context actually matter.

---

## Anti-Patterns (What NOT to Do)

❌ Sub-agent creates tmux session itself → exec may fail in sub-agent context
❌ Sub-agent spawns another sub-agent → infinite recursion loop
❌ Operator does the analysis itself → undermines Claude Code's purpose
❌ Kill tmux session after every task → destroys the one thing this skill preserves
❌ Poll every 60 seconds → interrupts Claude's long-running work
❌ Paste full task via tmux send-keys → fragile for long prompts
❌ Hard-code paths like /root/.openclaw/... → not portable across environments
