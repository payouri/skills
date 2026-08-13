---
name: orchestrate-backlog
description: Drain the AFK-ready backlog hands-off — every ready-for-agent issue implemented and reviewed by parallel sub-agents in isolated git worktrees, each parked on its own branch, then ask whether to merge. Use when the user points at a tracker rather than a single epic, or asks to drain the backlog.
argument-hint: "[max parallel tasks, default 4]"
---

# Orchestrate backlog

Sibling of [orchestrate-implementation](../orchestrate-implementation/SKILL.md). Same fleet
machinery, different unit of work and a different endpoint:

|             | orchestrate-implementation   | orchestrate-backlog (this skill)                |
| ----------- | ---------------------------- | ----------------------------------------------- |
| Input       | one epic issue               | the whole tracker, filtered by label            |
| Frontier    | open sub-tasks of that epic  | open `ready-for-agent` non-epic issues          |
| Concurrency | Workflow's own cap           | **4 tasks max**, unless the user says otherwise |
| Endpoint    | merged onto a feature branch | **parked**: one branch per issue                |
| Issue state | commented + closed           | commented, **left open and assigned**           |

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
- Read `git status --porcelain` in two halves. **Modified or staged tracked files: stop.** That work
  is the human's, a fleet running alongside it makes the result unreadable, and under
  [LANDING.md](LANDING.md) it also makes `merge --ff-only` refuse. **Untracked-only:** say what you
  see and proceed.
- Note the trunk SHA, and check whether `origin/<trunk>` is **ahead** of local trunk. If it is, every
  branch is about to be cut from a stale base and commits the fleet never authored will surface inside
  its diffs — say so before claiming anything.
- Snapshot `git worktree list`. Long-lived worktrees that aren't the fleet's are common, and the
  Sweep reconciles against this snapshot rather than assuming every checkout it finds is its own.
- Size the run to the wall clock: a `Workflow` run is cut off after roughly an hour, so plan for
  about **three tasks per run** and tell the user a large frontier takes several runs.
- Resolve `maxParallel`: 4 unless the user asked for a different number.
- No branch to create. Each task creates its own; there is no shared integration branch.

## 2. Run the workflow

Use the `Workflow` tool with the script in [REFERENCE.md](REFERENCE.md). It:

1. **Discovers** the frontier — open, unassigned, unblocked, non-`epic` issues carrying the
   AFK-ready label — and estimates each one's complexity from its issue text (`core` public API,
   `unsafe`, an ADR-covered area, or a wide file footprint → `standard`/Sonnet; narrow, low-risk,
   well-specified → `simple`/Haiku).
2. Runs each issue through **claim → implement → review-and-fix** in a hand-rolled lane pool, so at
   most `maxParallel` issues are in flight while lanes refill as tasks finish.
3. Loops rounds of discovery until the frontier is dry, picking up labels a human added mid-run and
   re-evaluating blockers against the issues this run already delivered.

Read REFERENCE.md before invoking — it has the script, the schemas, and the exact rules for the
concurrency pool, worktree lifecycle, model tiering, and HITL triggers. Adapt it to the repo's
tracker conventions rather than copying it verbatim.

If the user asks up front for each task to **land** — rebase onto trunk and fast-forward trunk to it
as it finishes — that is a different endpoint with rules of its own: read [LANDING.md](LANDING.md)
before adapting the script. Landing is also what §5 offers after a parked run; same rules, different
entry point.

## 3. Hard rules the script must enforce

- **Park, don't land.** Each issue ends as one squashed commit on its own `afk/issue-<n>` branch, and
  that branch is the deliverable. No sub-agent integrates — whether anything merges is a decision you
  put to the user in §5, once they can see what's on the branches.
- **Leave every issue open and assigned.** Comment with the branch, the commit SHA, and a short
  summary. Open-and-assigned _is_ the "done, awaiting human merge" state, and the assignee is what
  keeps a later round from re-claiming it.
- **An unclosed blocker starves its dependents.** Where the tracker models blocking as native issue
  dependencies, `blocked_by` never clears, so Discover must be handed the issues this run already
  delivered and told to treat them as satisfied. Otherwise the round loop only re-finds round 1 and
  reports a dry frontier over a backlog it silently dropped.
- **The implementer's trailer is `Refs #<n>`.** `Closes`/`Fixes`/`Resolves` closes the issue the
  moment it reaches the default branch.
- **Implementer never reviews its own work.** It implements via `/implement` (skipping that skill's
  built-in "now run /code-review" step), writes a handoff via `/handoff`, and stops.
- **Reviewer is a different agent, always on Opus**, starting only from the implementer's handoff.
  `/code-review`'s Spec axis _is_ the per-issue discrepancy check, so there is no separate
  discrepancy stage; it fixes findings itself, and the implementer gets no second pass.
- **At most `maxParallel` (default 4) issues in flight**, via the script's explicit pool.
  `pipeline()`/`parallel()` alone would run up to Workflow's own cap of ~16.
- **One worktree per issue**, reused across implement → review → fix, never shared between two
  concurrent issues — a worktree can only have one branch checked out.
- **The run ends as branches and nothing else.** Every exit path — success _and_ HITL — commits to the
  branch and disposes of the checkout. **Residue** becomes a marked `chore(wip)` commit on the branch,
  reported as unreviewed; a `git worktree remove` refusal means residue is still uncommitted, so go
  commit it. Branches survive the run; they're the deliverable.
- **The Sweep verifies that invariant rather than trusting it.** Every claim registers its worktree;
  the Sweep reconciles those against pre-flight's snapshot, recovers stragglers' residue onto their
  branches, and disposes of what's left. It runs even when every lane failed — that's when the most is
  stranded.
- **HITL escalation, not a workaround**, when an issue hits a security or destructive-action concern,
  or a conflict the reviewer can't resolve after a genuine attempt. Stop _fixing_, record why, keep the
  rest of the fleet going — but still commit and still dispose.
- **A dead agent degrades one lane, never the fleet — and know which death it is.** `agent()` returns
  `null` rather than throwing, so null-check every stage inside the lane and let nothing after the pool
  dereference an unchecked value. An **overload** kills every in-flight agent at once and wants
  escalating backoff, honoring any `Retry-After` the provider gave. A **deadline** — one aborted agent
  near the hour mark, every other lane complete — wants a reset, not a retry; backing off there only
  spends what's left of the run asleep. REFERENCE.md's `tryAgent` wrapper is not optional.

## 3b. If the run fails partway

Don't restart — recover. `journal.jsonl` in the run's transcript dir records each agent's real return
value, so it tells you which stages actually completed; then inspect each worktree for **residue**.
Both halves of the fleet die dirty in different ways — a killed reviewer leaves fixes atop a committed
implementation, a killed implementer leaves residue and an _empty branch_ — and residue must be
committed to its branch before a resume re-runs live into that tree.

A **deadline** is not a resume: the aborted issue is left assigned, with an empty branch and a live
worktree, and all three block a retry. Reset it and let the next run rediscover it.

Procedures for both in [REFERENCE.md](REFERENCE.md).

## 4. Final report

Once the workflow returns, give the user exactly four sections:

- **Branches ready to land** — one line per issue: `#<n> <title>` → branch, commit SHA, one-line
  summary. This is the main deliverable; the user integrates from it. Flag any branch carrying a
  `chore(wip)` residue commit — that part is unreviewed.
- **Spec discrepancies** — per issue, anything the reviewer found specified-but-undelivered or
  delivered-but-unspecified.
- **HITL escalations** — every issue whose review stopped early and why, or "none". Their work is on
  their branch too; say what state it's in.
- **Summary** — issues attempted, models used, rounds run, whether the round cap or the **wall clock**
  ended the run, and the Sweep result. State plainly whether `git worktree list` is back to
  pre-flight's snapshot; if anything survived, name the paths rather than implying a tidy finish. Name
  any issue the run never reached, and any left aborted-and-reset. If earlier runs left branches now
  fully merged into trunk, say so and offer `git branch -d` — safe by construction, since it refuses
  anything unmerged — rather than letting them accumulate.

## 5. Ask whether to integrate

The run leaves branches, and a pile of unmerged branches is unfinished business. Don't leave the user
to open that question themselves, and don't answer it for them: after the report, **ask**.

Ask _after_ the four sections, never before — the answer depends on the discrepancies and escalations
they just read, so asking first is asking them to decide blind. Put the merge decision as the question,
with landing all branches, landing a subset, and leaving everything parked as the options. Name what
makes the decision non-obvious: any HITL escalation, any `chore(wip)` residue commit, and the fact
that the branches were each cut from the same trunk tip and never tested against each other, so they
may well conflict.

If they say land, follow [LANDING.md](LANDING.md) — the same rules, entered from here instead of from
the script. Post-run there is no pool to race, so take the branches one at a time, each in its own
sub-agent: rebase onto current trunk, re-run the validation gate, fast-forward, close the issue. A
branch whose gate fails after a genuine attempt stays parked and is reported, and the remaining
branches still get their turn. Never push.

If they say park, say so plainly and stop. Parked is a finished state, not a failure.
