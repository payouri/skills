---
name: orchestrate-implementation
description: Orchestrate implementation of an epic/task and its open sub-tasks end-to-end using parallel sub-agents in isolated git worktrees, with strict implement/review separation, per-tier model selection, and HITL escalation on risk. Use when the user wants a multi-sub-task epic driven to completion hands-off, says "orchestrate this", "implement all the open sub-tasks in parallel", or points at an epic issue with open children.
argument-hint: "<epic issue number or URL>"
---

# Orchestrate implementation

You are the **orchestrator**. You never write or edit code yourself, and you never review code
yourself — every implementation, review, and fix happens inside a sub-agent spawned by the
`Workflow` tool. Your job is to resolve the epic into sub-tasks, run the workflow, and turn its
result into the final report.

If you catch yourself reaching for `Edit`/`Write`/`Bash -c "git commit"` on the target repo, stop —
that work belongs in an `agent()` call inside the script, not in this session.

## 1. Resolve the tracker and the epic

- Read `docs/agents/issue-tracker.md` at the target repo's root if it exists — follow whatever
  convention it documents (sub-issue API, checklist fallback, claim/close commands). If the file is
  absent, default to plain GitHub issues via `gh` (epic = one issue, sub-tasks = child issues linked
  in the body, no native sub-issue assumption).
- Resolve `<task>` to a single epic/parent issue. If the user gave a bare description instead of an
  issue reference, ask them for the issue number before proceeding — don't invent one.

## 2. Run the workflow

Use the `Workflow` tool with the script in [REFERENCE.md](REFERENCE.md), filling in `args` with the
epic number, repo slug, and a feature branch name (create the feature branch off `trunk` first if it
doesn't exist yet — this is a local branch, not a push). The script:

1. **Discovers** open, unblocked, unassigned sub-tasks and estimates each one's complexity/blast
   radius directly from its issue text (touches `core` public API, `unsafe`, an ADR-covered area, or
   a wide file footprint → `standard`/Sonnet; a narrow, low-risk, well-specified change → `simple`/Haiku).
2. Runs each sub-task through **claim → implement → review-and-fix** as a `pipeline()`, so sub-tasks
   at different stages overlap instead of lock-stepping.
3. Loops rounds of discovery until no eligible sub-tasks remain (finishing a sub-task can unblock
   another).
4. Ends with a **discrepancy-check** agent comparing the epic's spec against what actually landed.

Read REFERENCE.md before invoking — it has the full script, the schemas, and the exact rules for
worktree lifecycle, model tiering, commit shape, and HITL triggers. Adapt it to the repo's tracker
conventions rather than copying it verbatim.

## 3. Hard rules the script must enforce

- **Implementer never reviews its own work.** It implements via `/implement` (skip its built-in
  "now run /code-review" step), writes a handoff via `/handoff`, and stops.
- **Reviewer is a different agent, always on Opus**, and only starts from the implementer's handoff.
  It runs `/code-review` and fixes what it finds — the implementer doesn't get a second pass.
- **One worktree per sub-task, reused across implement → review → fix**, removed once the sub-task's
  branch has landed on the feature branch. Don't share a worktree between two sub-tasks running
  concurrently.
- **One commit per sub-task lands on the feature branch.** Intermediate fix commits from the review
  loop get squashed before merge — the feature branch is local and unpublished, so squashing here
  doesn't conflict with this repo's "never amend published commits" rule.
- **Merge to the feature branch is rebase-then-`--ff-only`**, same discipline as this repo's trunk
  workflow, just one level down.
- **Never push, open a PR, or merge the feature branch into `trunk`** as part of this skill — that's
  a separate, explicitly human-confirmed step.
- **HITL escalation, not a forced merge**, when a sub-task hits a security/destructive-action concern
  or a rebase/merge conflict the reviewer can't resolve after a genuine attempt. Leave the issue open,
  record why, keep going with the rest of the fleet.
- **Claim by assignee, close by comment-then-close**, per the tracker convention resolved in step 1.

## 4. Final report

Once the workflow returns, give the user exactly three sections:

- **Discrepancies** — epic spec vs. what was delivered, from the workflow's discrepancy-check stage.
- **HITL escalations** — every sub-task that stopped short of landing and why, or "none" if the fleet
  finished clean.
- **Summary** — sub-tasks landed, models used, and any rounds that hit the discovery-loop safety cap.
