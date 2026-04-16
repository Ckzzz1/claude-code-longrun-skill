---
name: claude-code-longrun
description: Use for Claude Code tasks that are long-running, iterative, or need persistent context across multiple rounds. Trigger when the user asks to use Claude Code for a complex coding task, wants tmux-based persistence, wants to reuse Claude Code context between follow-ups, or when a one-shot `claude --print` flow would likely lose too much context. Prefer this over generic coding-agent guidance for Claude Code sessions expected to run for many minutes or across multiple interactions.
---

# Claude Code Longrun

Use this skill when **Claude Code** is the requested tool and the task is too large, too iterative, or too context-heavy for a one-shot `claude --print` call.

## Use this skill when

- The user explicitly wants **Claude Code**.
- The task is expected to run for many minutes.
- You will likely send follow-up instructions into the same Claude Code session.
- The prompt is long enough that putting it directly on the command line is awkward.
- The job benefits from persistent working context, scratch notes, or multiple review and fix rounds.

## Do not use this skill when

- The task is tiny and can finish in one `claude --print` call.
- The user asked for Codex, Pi, ACP harness, or another tool.
- The work is a simple local edit you can do directly.
- The work must happen in a Discord thread via ACP harness. In that case use an ACP-specific workflow instead of this skill.

## Core model

Default to an **owner + operator** workflow.

- **Owner**: the agent coordinating the job
- **Operator**: a subagent that operates the existing tmux-backed local Claude Code session

This is the default because it keeps the workflow consistent:

1. the owner handles goals, boundaries, and reuse decisions
2. the operator subagent handles routine terminal interaction and monitoring
3. Claude Code remains the main worker inside tmux

Do not treat owner + operator as a license to add more layers. Prefer one owner and one operator.

## Why this is the default

`claude --print` is fine for short tasks, but it is not the best default when:

- you want to preserve context across multiple turns,
- the prompt is long,
- the task may branch into investigation, edits, tests, and fixes,
- or the user will probably say "continue", "再改一下", or "顺手把这个也做了".

A single owner can do everything, but longrun often includes a lot of repetitive terminal work. Splitting strategy from terminal operation usually gives a cleaner workflow, as long as responsibilities stay strict.

This skill is about orchestrating the local `claude` CLI, not routing work through ACP by default.

## Responsibilities

### Owner responsibilities

The owner should normally:

- decide that this task needs longrun instead of one-shot execution
- create or choose the tmux session name
- write the task file
- define scope, constraints, and success criteria
- pass the existing session name and task context to the operator
- decide whether to preserve, reuse, restart, or clean up the session
- communicate meaningful progress or final results back to the human

### Operator responsibilities

The operator should normally:

- be spawned as an OpenClaw `runtime:"subagent"` helper when delegation is useful
- use the existing tmux session created or designated by the owner
- launch the local Claude Code CLI inside that session
- send the short instruction telling Claude Code to read the task file
- monitor progress at low frequency
- send small execution-level follow-up instructions when the next step is obvious
- report blockers, ambiguity, or drift back to the owner quickly

### Operator autonomy

The operator may make small workflow decisions without escalating every time.

Examples:

- Claude Code is clearly still working, so wait and check later
- Claude Code needs a short execution-level nudge that stays within the written task boundaries
- Claude Code needs a brief reminder to keep changes minimal, run tests, or summarize risks

The operator should escalate to the owner when:

- the task goal appears ambiguous
- Claude Code proposes a tradeoff that needs judgment
- the work is drifting outside scope
- the session looks unhealthy or badly confused
- cleanup vs preservation is not obvious
- a user-facing decision is needed

## Hard boundaries

The operator must not:

- create nested tmux or nested longrun layers unless the owner explicitly wants that
- silently spawn more helper agents for the same job
- bypass Claude Code and independently do the main coding task
- change the task goal or scope on its own
- kill a still-useful session on its own

Do not require ACP for this workflow unless the environment has an explicitly configured ACP target and you intentionally want an ACP-based variant.

The owner must not:

- forget which session belongs to the job
- offload high-level decision making to the operator by accident

## Preferred workflow

### 1. Owner creates a dedicated tmux session

Use a clear session name related to the task.

Example:

```bash
tmux new-session -d -s claude-auth-fix
```

### 2. Owner writes a task file

Do not stuff a long brief directly into shell quoting if you can avoid it.

Create a task file inside the target project or a temp path, for example:

- `./.openclaw-tasks/claude-task.md`
- `/tmp/claude-task-<slug>.md`

The task file should include:

- the user goal
- scope boundaries
- constraints
- repo or path context
- required checks or tests
- expected final output format

Keep task files free of unnecessary secrets. Prefer describing required credentials or environment assumptions instead of pasting tokens, private keys, or personal data.

See `references/task-file-template.md` for a reusable structure.

### 3. Owner hands session + task context to operator

Pass the operator:

- the session name
- the working directory
- the task file path
- any important constraints or escalation rules

Preferred delegation path:

- spawn an OpenClaw subagent
- let that subagent operate tmux and the local `claude` CLI
- keep the owner responsible for strategy and lifecycle decisions

### 4. Operator launches Claude Code inside tmux

Preferred pattern:

```bash
cd /path/to/project && claude --permission-mode bypassPermissions
```

### 5. Operator tells Claude Code to read the task file

Preferred pattern: do **not** paste the entire task file contents into tmux if you can avoid it.

Send a short instruction such as:

```text
Read ./.openclaw-tasks/claude-task.md, follow it exactly, keep notes concise, and tell me when you are blocked or fully done.
```

### 6. Operator monitors with low interruption

Use `tmux capture-pane` occasionally. Let Claude Code work unless there is a real reason to intervene.

### 7. Owner decides session fate deliberately

Default bias: preserve context if follow-up reuse is likely.

Clean up only when:

- the task is clearly finished and reuse is unlikely,
- the session has gone bad and needs replacement,
- or the user explicitly wants cleanup.

## Monitoring strategy

### Default cadence

Check output with `tmux capture-pane` at low frequency.

Good default cadence:

- first check after a short settling period
- then about every 5 minutes for long tasks
- faster only when Claude Code is clearly waiting for input or repeatedly failing

### Look for

- explicit questions
- blockers
- test failures
- evidence of drift
- clear completion

### Avoid

- interrupting just because output paused briefly
- resending the task repeatedly
- turning the operator into a chatty middle manager
- killing and restarting a healthy run too early

## Decision guide

### Use one-shot `claude --print` when

- the ask is narrow
- no persistent context is needed
- a single response is likely enough

### Use this skill when

- the ask is open-ended or investigative
- the user is likely to iterate on Claude Code's work
- you expect code, test, revise, and re-run cycles
- preserving Claude Code's local context is part of the value

## Output expectations

When using this skill, keep the human updated only when something meaningful changes:

- started and where it is running
- blocked and what input is needed
- milestone reached
- finished with concrete results

Do not spam the user with routine polling updates.

## Common pitfalls

- Starting Claude Code with a huge inline prompt instead of a task file
- Polling too often and mistaking quiet work for failure
- Forgetting to run from the correct project directory
- Letting helper agents create extra layers without clear ownership
- Letting the operator make strategy decisions it should have escalated
- Killing a healthy session too early and losing follow-up context
- Using this pattern for tiny tasks that should have been one-shot
- Embedding local machine assumptions or private workflow details that make the skill less portable

## References

- Task file template: `references/task-file-template.md`
- Operation examples: `references/operation-examples.md`
