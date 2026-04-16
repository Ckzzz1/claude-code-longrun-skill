# Claude Code Longrun Operation Examples

These are practical patterns for running the local Claude Code CLI as a persistent tmux-backed worker under the default **owner + operator** model.

## Principle

Use one clear owner and one operator.

- The owner creates or chooses the session.
- The owner writes the task file.
- The operator is usually an OpenClaw subagent and uses that existing session.
- The operator handles routine terminal interaction.
- Claude Code does the main task.
- The owner decides whether the session should be preserved or cleaned up.

This is not an ACP-first workflow.

## Example 1: Owner starts a long task

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
```

Then pass `SESSION`, `PROJECT`, and `TASK_FILE` to the operator.

In practice, that operator is usually an OpenClaw `runtime:"subagent"` helper.

## Example 2: Operator launches Claude Code in the existing session

```bash
PROJECT=~/projects/my-repo
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" "cd '$PROJECT' && claude --permission-mode bypassPermissions" Enter
sleep 2

tmux send-keys -t "$SESSION" -l -- "Read ./.openclaw-tasks/auth-fix.md and execute it. If blocked, stop and ask one clear question."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

## Example 3: Operator polls without over-interrupting

Use when the task is already running and you only want to check progress.

```bash
SESSION=claude-my-repo-auth

tmux capture-pane -t "$SESSION" -p | tail -40
```

If Claude Code is actively investigating, editing, or testing, leave it alone.

A good default is to check roughly every 5 minutes for longer tasks.

## Example 4: Operator sends a small corrective nudge

Use when Claude Code is blocked or needs a narrow execution-level clarification that does not change task scope.

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" -l -- "Use the existing retry contract. Do not introduce a new config flag. Continue with the minimal fix."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

If the right move is not obvious, escalate to the owner instead of improvising.

## Example 5: Owner reuses the same session in a follow-up turn

Use when the user says something like "继续", "顺手把测试也补了", or "再把日志整理一下".

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" -l -- "Continue in the same branch. Add or update targeted tests for the refresh flow, then summarize what changed."
sleep 0.1
tmux send-keys -t "$SESSION" Enter
```

This is the main reason to prefer tmux over a one-shot `claude --print` flow.

## Example 6: Inspect full scrollback when recent output is not enough

```bash
SESSION=claude-my-repo-auth

tmux capture-pane -t "$SESSION" -p -S -
```

Use this sparingly. Usually the last 30 to 60 lines are enough.

## Example 7: Stop a clearly bad run

Only do this when Claude Code is truly stuck, looping, or working on the wrong thing, and the owner agrees or has already set that rule.

```bash
SESSION=claude-my-repo-auth

tmux send-keys -t "$SESSION" C-c
```

Then either:

- send a corrected instruction into the same session, or
- start a fresh session if the current context is too polluted

## Example 8: Clean up only when reuse is no longer useful

Use when the task is truly done and you do not expect another follow-up in the same session.

```bash
SESSION=claude-my-repo-auth

tmux kill-session -t "$SESSION"
```

Do not make this the automatic default for every longrun.

## Example 9: Tiny task, do not use tmux

For a narrow one-shot ask, use `claude --print` instead.

```bash
cd ~/projects/my-repo && claude --permission-mode bypassPermissions --print 'Explain where the refresh token is validated and list the relevant files.'
```

Use the longrun skill only when persistence and follow-up context actually matter.
