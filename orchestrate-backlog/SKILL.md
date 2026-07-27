---
name: orchestrate-backlog
description: Orchestrate implementation of every ready-for-agent issue in the tracker using parallel sub-agents in isolated git worktrees, capped at 4 concurrent tasks, with strict implement/review separation, per-tier model selection, and HITL escalation on risk. Each issue lands on its own branch and nothing is merged, pushed, or closed — integration stays human. Use when the user wants the AFK-ready backlog drained hands-off, says "implement all the ready-for-agent issues", "drain the backlog", "orchestrate the ready tasks", or points at a tracker rather than a single epic.
argument-hint: "[max parallel tasks, default 4]"
---

# Orchestrate backlog

Sibling of [orchestrate-implementation](../orchestrate-implementation/SKILL.md). Same fleet
machinery, different unit of work and a different endpoint:

|              | orchestrate-implementation      | orchestrate-backlog (this skill)          |
| ------------ | ------------------------------- | ----------------------------------------- |
| Input        | one epic issue                  | the whole tracker, filtered by label      |
| Frontier     | open sub-tasks of that epic     | open `ready-for-agent` non-epic issues    |
| Concurrency  | Workflow's own cap              | **4 tasks max**, unless the user says otherwise |
| Endpoint     | merged onto a feature branch    | **one branch per issue, nothing merged**  |
| Issue state  | commented + closed              | commented, **left open and assigned**     |

You are the **orchestrator**. You never write, edit, or review code yourself — every
implementation, review, and fix happens inside a sub-agent spawned by the `Workflow` tool. Your job
is pre-flight, running the workflow, and turning its result into the final report. If you catch
yourself reaching for `Edit`/`Write`/`git commit` on the target repo, stop: that work belongs in an
`agent()` call inside the script.

## 1. Pre-flight

- Read `docs/agents/issue-tracker.md` and `docs/agents/triage-labels.md` at the target repo's root.
  Follow whatever conventions they document — in particular the **actual label string** for the
  AFK-ready triage role, which may not literally be `ready-for-agent`. If those files are absent,
  default to plain GitHub issues via `gh` and the literal labels `ready-for-agent` / `epic`.
- Confirm the primary checkout is clean (`git status --porcelain` empty) and note the trunk SHA.
  Uncommitted work here is the human's, and a fleet running alongside it makes the result
  unreadable — stop and say so rather than working around it.
- Resolve `maxParallel`: 4 unless the user asked for a different number in the prompt.
- No branch to create. Each task creates its own; there is no shared integration branch.

## 2. Run the workflow

Use the `Workflow` tool with the script in [REFERENCE.md](REFERENCE.md). It:

1. **Discovers** the frontier — open, unassigned, unblocked, non-`epic` issues carrying the
   AFK-ready label — and estimates each one's complexity from its issue text (`core` public API,
   `unsafe`, an ADR-covered area, or a wide file footprint → `standard`/Sonnet; narrow, low-risk,
   well-specified → `simple`/Haiku).
2. Runs each issue through **claim → implement → review-and-fix** in a hand-rolled 4-lane pool, so
   at most `maxParallel` issues are in flight while lanes refill as tasks finish.
3. Loops rounds of discovery until the frontier is dry (a landed task can unblock another, and a
   human can add a label mid-run).

Read REFERENCE.md before invoking — it has the script, the schemas, and the exact rules for the
concurrency pool, worktree lifecycle, model tiering, and HITL triggers. Adapt it to the repo's
tracker conventions rather than copying it verbatim.

## 3. Hard rules the script must enforce

- **Nothing is merged.** No rebase onto trunk, no `--ff-only`, no push, no PR. Each issue's work
  ends as one squashed commit on its own `afk/issue-<n>` branch, and that is the deliverable.
  A sub-agent that "helpfully" merges has broken the skill's contract.
- **Nothing is closed.** The reviewer comments on the issue with the branch name, commit SHA, and a
  short summary, then leaves it **open and still assigned**. Open-and-assigned is the "done, awaiting
  human merge" state, and the assignee is also what stops a later round re-claiming it.
- **Implementer never reviews its own work.** It implements via `/implement` (skipping that skill's
  built-in "now run /code-review" step), writes a handoff via `/handoff`, and stops.
- **Reviewer is a different agent, always on Opus**, starting only from the implementer's handoff.
  It runs `/code-review` — whose Spec axis is also where per-issue discrepancy checking happens, so
  there is no separate discrepancy stage — and fixes findings itself. The implementer gets no second pass.
- **At most `maxParallel` (default 4) issues in flight.** The pool is explicit in the script;
  `pipeline()`/`parallel()` alone would run up to Workflow's own cap of ~16.
- **One worktree per issue, reused across implement → review → fix.** Never shared between two
  concurrent issues — a worktree can only have one branch checked out.
- **Worktree removed, branch kept.** Every worktree is disposed once its commit exists
  (`git worktree remove` + `git worktree prune`); the branch survives for the human. Removal requires
  a clean worktree, which is the check that no work is being thrown away — never `--force` past it.
- **HITL escalation, not a workaround**, when an issue hits a security/destructive-action concern, or
  a rebase/conflict the reviewer can't resolve after a genuine attempt. Leave the issue open, record
  why, keep the rest of the fleet going.
- **A dead agent degrades one lane, never the fleet.** `agent()` returns `null` rather than throwing
  when a subagent dies, and a provider-side overload kills every in-flight agent at once. Null-check
  every stage inside the lane, let nothing after the pool dereference an unchecked value, and back off
  with escalating waits (honoring any `Retry-After` the provider gave) instead of re-spawning
  immediately into the same overload. REFERENCE.md's `tryAgent` wrapper is not optional.

## 3b. If the run fails partway

Don't restart — recover. Read `journal.jsonl` in the run's transcript dir to see what actually
completed, inspect each worktree for uncommitted work a killed reviewer left behind, then
`Workflow({ scriptPath, resumeFromRunId })` so completed stages replay from cache. Keep surviving
prompts byte-identical or they re-run live. Full procedure in [REFERENCE.md](REFERENCE.md).

## 4. Final report

Once the workflow returns, give the user exactly four sections:

- **Branches ready to land** — one line per issue: `#<n> <title>` → branch, commit SHA, one-line
  summary. This is the main deliverable; the user integrates from it.
- **Spec discrepancies** — per issue, anything the reviewer found specified-but-undelivered or
  delivered-but-unspecified.
- **HITL escalations** — every issue that stopped short of a commit and why, or "none".
- **Summary** — issues attempted, models used, rounds run, whether the round cap was hit, and any
  worktree or branch left behind that the user should sweep.
