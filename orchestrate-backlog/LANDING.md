# Landing mode

The default endpoint is **park**: each issue ends as one commit on its own branch. Landing — rebase
onto trunk, fast-forward trunk to the branch tip — is reached two ways, and the rules below hold for
both:

- **In-run**, when the user asks up front ("_when a task is done, rebase it over trunk and fast-forward
  trunk_"). That replaces the endpoint, so say once that it overrides [SKILL.md](SKILL.md)'s park rule,
  then build it into the script. Landing happens inside each lane, and **must be serialized**.
- **Post-run**, when [SKILL.md](SKILL.md) §5 asks and the user says land. The fleet is finished, so
  there is no pool to race: walk the branches one at a time, one sub-agent each, in the order they were
  produced. Skip the mutex — sequence is the lock. Everything else applies unchanged.

Three rules make either path safe. Each is the fix for a specific way landing corrupts trunk.

## Serialize landing (in-run only)

Two lanes fast-forwarding trunk concurrently is a lost-update race. Put the whole sequence — rebase →
validate → `--ff-only` → comment → close → dispose — behind a mutex, so exactly one task is
integrating at any moment. Lanes still implement and review concurrently; only integration is
single-file.

A promise chain is enough, and it survives a stage that throws:

```js
let landLock = Promise.resolve()
function withLandLock(fn) {
  const run = landLock.then(fn, fn)
  landLock = run.then(
    () => {},
    () => {}
  )
  return run
}
```

Tell the landing agent it holds an exclusive lock and is the only agent allowed to touch trunk right
now. Without that, an agent that finds trunk unexpectedly moved starts inventing recovery.

## Rebase onto current trunk, then re-validate

A task's claim-time base is stale by the time it lands, because earlier tasks in the same run already
moved trunk. So rebase onto trunk's **current** tip, not the recorded base, and expect conflicts
between tasks that touch the same surface — resolve them via `/resolving-merge-conflicts`, keeping the
intent of both sides.

Then re-run the repo's full validation gate **after** the rebase. This is the point of the exercise: a
rebase onto another lane's work breaks code that passed in isolation, and only a post-rebase gate
catches it. If the gate still fails after a genuine fix attempt, or the conflict is unresolvable,
`git rebase --abort` so the branch is left intact, report `landed=false` with HITL, and land nothing.
A red trunk costs the human more than an unlanded branch.

Fast-forward from the primary checkout: `git -C <primary> merge --ff-only <branch>`. It must be a
fast-forward — you just rebased, so a refusal means something moved underneath you. Re-check trunk and
redo the rebase rather than substituting a merge commit or a reset. Never push: the remote stays the
human's, and a local trunk is cheap to reset if they dislike a landing.

## Close on land, and only on land

Once the work is on trunk, open is a lie. Closing is also what releases the dependents that
[SKILL.md](SKILL.md)'s starvation rule warns about — landing without closing leaves every dependent
issue at `blocked_by > 0` forever, so the round loop never discovers them and the run reports a dry
frontier over work it dropped.

Gate it on the fast-forward actually happening. A task that reported `landed=false` for any reason
stays open and assigned, exactly as in park mode.

## What doesn't change

Everything else in [SKILL.md](SKILL.md) holds. The reviewer still never integrates — landing is a
separate stage after review, and a reviewer that rebases has raced the lock. Residue still becomes a
`chore(wip)` commit rather than being discarded. The Sweep still runs. Branches still survive the run:
after landing they're merged into trunk, which is what makes `git branch -d` safe to offer in the
final report.

The report's **Branches ready to land** section becomes **Landed on trunk** — one line per issue with
its commit SHA, the resulting trunk SHA, and what the rebase had to resolve. Say plainly how far ahead
of `origin` trunk now sits, since nothing was pushed.
