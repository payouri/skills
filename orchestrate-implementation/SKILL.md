---
name: orchestrate-implementation
description: Orchestrate implementation of an epic/task and its open sub-tasks end-to-end using parallel sub-agents in isolated git worktrees, then red-team the assembled feature branch with adversarial-code-review and fix every surviving defect at the right model tier. Use when the user wants a multi-sub-task epic driven to completion hands-off, says "orchestrate this", "implement all the open sub-tasks in parallel", or points at an epic issue with open children.
argument-hint: "<epic issue number or URL>"
---

# Orchestrate implementation

You are the **orchestrator**. You never write, review, or fix code yourself — every implementation,
integration, review, and fix happens inside a sub-agent. Your job is to resolve the epic into
sub-tasks, drive three acts in sequence, and turn their results into the final report.

If you catch yourself reaching for `Edit`/`Write`/`Bash -c "git commit"` on the target repo, stop —
that work belongs in an `agent()` call inside a script, not in this session.

The run is **three acts, orchestrator in the loop between each**:

| Act              | What runs                                                              | Who reviews                |
| ---------------- | ---------------------------------------------------------------------- | -------------------------- |
| **1. Build**     | Workflow A — discover → claim → implement → integrate, looped           | nobody                     |
| **2. Attack**    | `/adversarial-code-review` over the whole feature branch, all five lenses | 5 attackers + 5 refuters |
| **3. Repair**    | Workflow B — tiered fixers for every non-refuted finding, discrepancy check, then the epic comment | nobody |

Review is **branch-wide and once**, not per-sub-task. An implementer implements and stops; nothing
reviews its diff in isolation. That is deliberate — a defect that exists only because sub-task #12
and sub-task #17 landed together is invisible to a per-sub-task reviewer, and is exactly what act 2
is for.

## 0. Resolve the tracker, the epic, and the branch

- Read `docs/agents/issue-tracker.md` at the target repo's root if it exists — follow whatever
  convention it documents (sub-issue API, checklist fallback, claim/close commands). If the file is
  absent, default to plain GitHub issues via `gh` (epic = one issue, sub-tasks = child issues linked
  in the body, no native sub-issue assumption).
- Resolve `<task>` to a single epic/parent issue. If the user gave a bare description instead of an
  issue reference, ask them for the issue number before proceeding — don't invent one.
- Resolve the **base branch** — whatever this repo integrates into, and whatever it calls it: `trunk`,
  `main`, `master`, a release branch, or the epic's own parent branch if the work nests. Read it from
  the repo's docs or `git symbolic-ref refs/remotes/origin/HEAD`; never assume the name.
- Create the feature branch off the base branch if it doesn't exist (local, never pushed) and **check
  it out in the primary repo**. It stays checked out there for the whole run: the integrator merges
  into it, and acts 2–3 diff against it. Sub-tasks and fixes live in worktrees; nothing but the
  integrator writes to the primary checkout.
- **Pin the fixed point now, as a SHA**: `git rev-parse <featureBranch>` before a single sub-task
  lands. That commit — call it `baseSha` — is what act 2 reviews against and act 3 measures against,
  and it is the one thing that stays correct no matter what the base branch is named or how it moves
  during the run. A branch name as the fixed point silently changes meaning if someone lands on the
  base mid-run; a SHA cannot. If you're resuming and the branch already has commits on it, recover the
  pin with `git merge-base <baseBranch> <featureBranch>` instead — never re-pin to the branch's current
  tip, which would hide everything already landed from the review.

## 1. Act 1 — build

Run Workflow A from [REFERENCE.md](REFERENCE.md) § Workflow A. It:

1. **Discovers** open, unblocked, unassigned sub-tasks and estimates each one's complexity/blast
   radius from its issue text (touches `core` public API, `unsafe`, an ADR-covered area, or a wide
   file footprint → `standard`/Sonnet; a narrow, low-risk, well-specified change → `simple`/Haiku).
2. Runs **claim → implement** as a `pipeline()`, so sub-tasks at different stages overlap. The
   implementer commits one squashed commit on its own `subtask-<n>` branch and **stops there** — it
   does not review, does not merge, does not close.
3. Barriers, then runs **one serial integrator** that rebases and `--ff-only` merges each sub-task
   branch onto the feature branch one at a time, comments and closes each issue as it lands, and
   removes the worktree. Serial because N agents racing one branch ref is how you get lost updates
   and phantom conflicts.
4. Loops rounds of discovery until no eligible sub-tasks remain (landing one can unblock another).

## 2. Act 2 — attack

Invoke `/adversarial-code-review` and follow its own SKILL.md §1–§4 against the assembled branch:

- **Fixed point**: the `baseSha` pinned in step 0 — not a branch name. `git diff <baseSha>...HEAD`
  from the primary checkout is the epic's whole contribution and nothing else.
- **Lenses**: all five — Bugs, Maintainability, Security, Standards, Spec. Pass them explicitly so
  its §2 `AskUserQuestion` does not fire.
- **Spec rulebook**: the epic issue's body and acceptance criteria, fetched. The Spec lens is never
  unavailable here — an epic always has a spec.

Don't re-implement its attack/refute logic. Run its script, hold its per-lens verdicts, and carry the
`REFUTED` ones through to the report's audit log.

## 3. Act 3 — repair

Run Workflow B from [REFERENCE.md](REFERENCE.md) § Workflow B, passing **every** finding act 2
produced — `REFUTED` and `UNJUDGED` included, since the script both filters for what to fix and
tallies the rest — plus **act 1's return value** as `build`. It:

1. **Triages each finding's _fix_ complexity** — not the defect's severity. A blocker fixed by
   flipping a comparison is `trivial`/Haiku; a minor smell whose fix reshapes an interface is
   `intricate`/Opus.
2. Groups findings by file so two fixers never contend for the same lines, fixes each group in its
   own worktree off the feature branch, then lands them through the same serial-integrator discipline.
3. Runs the **discrepancy check** — epic spec vs. what actually landed, measured against `baseSha`.
4. **Comments the merge-readiness report on the epic** — unconditionally, and it does not close it.

`REFUTED` findings are never fixed. `UNJUDGED` findings (the refuter died before verdicting them) are
**reported, never auto-fixed** — nothing has vetted them, so a fixer would be acting on an unchecked
claim.

### The epic comment

Everything a run produces — sub-tasks landed, five lenses of review, fixes, discrepancies — otherwise
exists only in the orchestrator's terminal, while the epic shows children silently closing one by one.
The comment is the durable record, addressed to whoever decides whether the branch is mergeable:

- **Composed in the script, not by the agent.** The body is built from the run's actual values; the
  posting agent is told to post it verbatim and may not summarize, reorder, or soften it. A summarizer
  handed raw results is free to round "one confirmed blocker unfixed" down to "minor issues remain".
- **Leads with what blocks a merge**: every non-refuted finding no fix landed for (severity orders the
  list, it doesn't gate membership — a survived `major` matters to the merger too), every `UNJUDGED`
  claim, every sub-task or fix that escalated or died, and where that work is still sitting.
- **Posted even on a clean run.** Silence can't distinguish "nothing outstanding" from "the fleet
  never got there", so a clean run says so explicitly.
- **Never closes the epic, never touches labels.** The epic closes when a human merges the branch,
  which this skill never does.

Act 1's integrator does the same at sub-task level: when a sub-task fails to land it comments why and
names the branch and worktree holding its work, rather than leaving an issue open and assigned, which
reads as in-progress and gets skipped by later rounds on the assignee alone.

## 4. Hard rules every act must enforce

- **Nobody reviews their own work, and nobody reviews per-sub-task.** Implementers implement.
  Attackers attack a branch they didn't write. Fixers fix findings they didn't raise.
- **Implementers never invoke `/code-review` or `/adversarial-code-review`.** Review is act 2's job,
  once, over the assembled branch. A per-sub-task review is wasted spend and blind to interaction
  defects.
- **One worktree per sub-task and per fix group**, removed once its branch has landed. Never shared
  between two units of work running concurrently.
- **Only the integrator writes to the feature branch**, and only serially. Rebase, then `--ff-only`.
- **The integrator never resolves a conflict.** On any conflict it aborts the rebase, leaves the
  branch unmerged, and reports HITL. A guessed resolution corrupts the branch silently; a stalled
  sub-task doesn't.
- **One commit per sub-task and per fix group lands on the feature branch.** Intermediate commits are
  squashed before merge — the feature branch is local and unpublished, so squashing here doesn't
  conflict with this repo's "never amend published commits" rule.
- **Never push, open a PR, or merge the feature branch into the base branch** — that's a separate,
  explicitly human-confirmed step.
- **HITL escalation, not a forced merge**, on a security/destructive-action concern or any conflict.
  Leave the issue open, **comment why**, keep going with the rest of the fleet. An escalation nobody
  wrote down is indistinguishable from work still in flight.
- **The run leaves a trace on the tracker, not just in the terminal.** The epic gets the
  merge-readiness comment; a stalled sub-task gets the reason it stalled. Neither gets closed.
- **Claim by assignee, close by comment-then-close** at land time, per the tracker convention from
  step 0. Findings that later touch a closed sub-task become fix commits, not reopened issues.
- **A dead agent degrades one unit of work, never the fleet.** `agent()` returns `null` rather than
  throwing when a subagent dies, and a provider-side overload kills every in-flight agent at once.
  Null-check every stage, let nothing outside `pipeline()` dereference an unchecked value, and back
  off with escalating waits (honoring any `Retry-After` the provider gave) instead of re-spawning
  straight back into the same overload. REFERENCE.md's `tryAgent` wrapper is not optional.

If a workflow dies partway, **recover rather than restart**: read `journal.jsonl` for what actually
completed, inspect each worktree for uncommitted work a killed agent left, then
`Workflow({ scriptPath, resumeFromRunId })` so completed stages replay from cache. Keep surviving
prompts byte-identical or they re-run live. Full procedure in [REFERENCE.md](REFERENCE.md).

## 5. Final report

Once act 3 returns, give the user exactly five sections:

- **Landed** — sub-tasks that reached the feature branch, with the model tier each used.
- **Review** — per-lens tally (`Bugs 2 confirmed / 1 plausible / 3 refuted · Security 0 / 0 / 2`),
  then every non-refuted finding and whether its fix landed. Never rerank findings across lenses:
  the separation is what stops a loud Maintainability haul from burying the one Bugs blocker.
- **Not fixed** — `UNJUDGED` findings, plus any fix that failed or escalated. These are the user's.
- **Discrepancies** — epic spec vs. what was delivered.
- **HITL escalations** — every unit of work that stopped short of landing and why, or "none".

Close with the epic comment's URL. If `epicComment.posted` is false, say so and paste
`epicComment.body` — the record has to land somewhere, and the terminal is the fallback, not the plan.
