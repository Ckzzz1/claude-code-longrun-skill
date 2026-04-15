# Claude Code Task File Template

Use this structure for long-running Claude Code jobs.

## Goal

State the end result in one or two sentences.

## Project Context

- repo or directory
- relevant files or subsystems
- anything Claude should inspect first

## Constraints

- tool or stack constraints
- files to avoid
- compatibility or style requirements
- whether destructive changes are forbidden

## What to do

List the requested work as concrete steps.

1. Investigate the current implementation.
2. Make only the necessary changes.
3. Run the appropriate checks.
4. Summarize what changed and any risks.

## What not to do

List tempting but out-of-scope work.

## Validation

Specify exact checks when possible.

- run tests
- run lint
- verify specific behavior

## Deliverable

Tell Claude what to report back with.

- files changed
- commands run
- result summary
- remaining risks or follow-ups

## Escalation

If blocked, stop and report:

- what is blocked
- why
- the smallest decision or input needed
