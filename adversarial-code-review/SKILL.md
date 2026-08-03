---
name: adversarial-code-review
description: Red-team a diff one lens at a time (Bugs, Maintainability, Security, Standards, Spec), then refute every finding before it reaches you.
disable-model-invocation: true
argument-hint: "<fixed point> [lenses]"
---

# Adversarial code review

Two hostilities, stacked.

First **attack**: one sub-agent per **lens**, each hunting the way the diff fails.
Then **refute**: a fresh sub-agent per lens that never sees the attacker's reasoning and tries to
kill every claim it made. Only what survives reaches the report.

You are the orchestrator. You pin the diff, resolve the lenses, run the workflow, and turn its
verdicts into the report — you never review the code yourself. Reviewing here means spawning a lens,
and a finding you produce yourself has no refuter.

## 1. Pin the fixed point

Whatever the user named is the fixed point — a SHA, branch, tag, `main`, `HEAD~5`. If they named
none, ask for it.

Then, before anything spawns: `git rev-parse <fixed-point>` resolves, and
`git diff <fixed-point>...HEAD --stat` is non-empty. A bad ref fails here, in one place, rather than
inside five sub-agents. Record the three-dot diff command and the output of
`git log <fixed-point>..HEAD --oneline`.

## 2. Demand the lenses

- **Bugs** — does it break?
- **Maintainability** — does it leave the codebase worse?
- **Security** — can it be abused?
- **Standards** — does it defy this repo's documented rules?
- **Spec** — does it do what was asked?

If the prompt named lenses, take those. Otherwise put the five to the user with `AskUserQuestion`
(`multiSelect: true`) and wait. The lens set decides the whole run's cost and is the user's call —
never inferred from the diff.

Done when the lens set is explicit and non-empty.

## 3. Resolve each lens's rulebook

The table in [REFERENCE.md](REFERENCE.md) § Lens rulebooks gives the resolution for all five. Turn
each chosen lens into the rulebook instruction string its attacker will receive.

Bugs, Maintainability, and Standards read rulebooks that ship in this skill's `rulebooks/`, and
Security invokes a bundled skill, so **Spec** is the only lens that can come up empty: no issue, PRD,
or spec found makes it unavailable — name that to the user and let them drop it or point you at the
spec. An attacker never infers the requirements it is meant to be checking against.

Spec is also the only lens whose rulebook can be **incomplete without looking it**. When the fixed
point spans several issues, the spec is the union of all of them — the parent states the goal, its
children carry the acceptance criteria the code was written against. Gather them all; REFERENCE.md
§ Lens rulebooks has the rules for assembling a composite spec.

Done when every chosen lens carries a rulebook instruction, or Spec has been dropped.

## 4. Attack, then refute

Use the `Workflow` tool with the script in [REFERENCE.md](REFERENCE.md) § Workflow script, `args`
shaped as documented there. It pipelines one attacker and one refuter per lens, so a lens is being
refuted while slower lenses are still attacking.

The wall between the two stages is mechanical: the script strips each finding's `rationale`, `fix`,
and `severity` before the claims reach the refuter, leaving only the claim, its location, the quoted
hunk, and the failure scenario. The refuter re-reads the code and reaches its own conclusion, so
keep the stripping in the script — a prompt asking a refuter to ignore reasoning it can already see
does not hold.

Done when every finding carries a verdict — `CONFIRMED`, `PLAUSIBLE`, or `REFUTED`. A finding the
script marks `UNJUDGED` means its refuter dropped it; say so in the report rather than promoting it.

## 5. Report

One section per lens that ran, `CONFIRMED` before `PLAUSIBLE`, blockers first within each group:

```
## Bugs
**CONFIRMED · blocker** — <claim> · `path:line`
Fails when: <concrete input or state → wrong outcome>
Fix: <the smallest change that removes it>
Confirmed because: <the refuter's one line>
```

Then the refuted log — one line each, the claim and what killed it:

```
## Refuted
- <claim> · `path:line` — <why it died>
```

That log is the gate's audit trail: an over-eager refuter is visible in it, where silent dropping
would hide a real bug behind a clean report.

Close with a per-lens tally and nothing more — `Bugs 2 confirmed / 1 plausible / 3 refuted ·
Security 0 / 0 / 2`. Findings are never reranked across lenses: the separation is what stops a loud
Maintainability haul from burying the one Bugs blocker.

Stop at the report. Survivors go to `/simplify`, `/implement`, or `/to-issues` — a fix phase in this
context would have the lenses triaging for fixability instead of attacking.
