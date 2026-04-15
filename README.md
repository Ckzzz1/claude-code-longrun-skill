# claude-code-longrun

A public AgentSkill for running Claude Code in a persistent, tmux-backed workflow for long, iterative coding tasks.

## What it does

This skill guides an agent to:

- start Claude Code in a dedicated tmux session
- place long instructions in a task file instead of shell quoting
- monitor progress at low frequency
- preserve context across follow-up requests

It is intended for tasks that are too long or iterative for a one-shot `claude --print` flow.

## Repository layout

```text
claude-code-longrun/
├── SKILL.md
└── references/
    ├── operation-examples.md
    └── task-file-template.md
```

## Compatibility

This repository is structured as an AgentSkill-style package.

It should work directly in systems that support `SKILL.md`-based skills or can ingest the packaged `.skill` archive format. In other agent frameworks, the contents may still be useful as reusable prompt and workflow guidance, but direct plug-and-play compatibility is not guaranteed.

## Safety and portability

This repo is prepared for public release:

- no personal contact info
- no secrets or API keys
- no machine-specific absolute paths
- examples use placeholder project/session names

Do not place secrets, tokens, or private keys inside task files created from this skill.

## Packaging

If your environment includes the OpenClaw skill tooling, package it with:

```bash
python /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/package_skill.py ./claude-code-longrun ./dist
```

## License

MIT
