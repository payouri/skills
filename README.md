# skills

A collection of agent skills.

## Available skills

- [orchestrate-implementation](orchestrate-implementation/SKILL.md) — Orchestrate implementation of an epic/task and its open sub-tasks end-to-end using parallel sub-agents in isolated git worktrees, with strict implement/review separation, per-tier model selection, and HITL escalation on risk.
- [orchestrate-backlog](orchestrate-backlog/SKILL.md) — Same fleet machinery aimed at the whole tracker instead of one epic: drains every open `ready-for-agent` issue, 4 tasks in flight at a time, landing each on its own branch without merging, pushing, or closing anything.
- [research-dossier](research-dossier/SKILL.md) — Compile any topic into an evidence-graded dossier (guide, rulebook, sources) filed under one directory of the research repo and summarised in its README.

The two orchestrators are siblings and share their failure-handling design — read both before changing either.

## Installing

This repo is the source of truth. Link each skill into your agent's skills directory rather than
copying it, so edits here take effect immediately:

```bash
ln -s "$PWD/orchestrate-implementation" ~/.claude/skills/orchestrate-implementation
ln -s "$PWD/orchestrate-backlog"        ~/.claude/skills/orchestrate-backlog
ln -s "$PWD/research-dossier"           ~/.claude/skills/research-dossier
```
