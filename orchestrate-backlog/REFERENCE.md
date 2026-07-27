# Workflow script template

Adapt this before calling `Workflow`. It's written against the plain-`gh` convention in
`docs/agents/issue-tracker.md` plus the label table in `docs/agents/triage-labels.md`; if the target
repo documents something else, change the prose inside the `agent()` prompts — the control flow
(discover → 4-lane pool → loop) stays the same.

Pass `args: { repo: "<owner>/<repo>", readyLabel: "ready-for-agent", epicLabel: "epic", maxParallel: 4 }`.
Add `retryAfterMs: <ms>` when a previous run surfaced a `Retry-After` header — it becomes the backoff floor.

> **Known issue — `args` may arrive JSON-stringified.** Observed repeatedly across separate projects
> and invocations: the object passed as `args` reaches the script as a *string* holding its JSON.
> `args` is the only `Workflow` input with no declared type, so nothing enforces object-ness, and on a
> string primitive every property read yields `undefined` — which then interpolates into prompts as the
> literal text `"undefined"`, silently, with no TypeError. The script normalizes for this
> (parse-if-string, below) and hard-fails on anything still missing. If you see it recur, **skip
> `args` entirely and inline the config as literals**:
>
> ```js
> const cfg = { repo: 'payouri/image_manipulation', readyLabel: 'ready-for-agent', epicLabel: 'epic', maxParallel: 4 }
> ```
>
> then destructure from `cfg` rather than `input`. Workflow persists a script per invocation and you
> iterate via `scriptPath` anyway, so a per-run literal costs nothing in reusability.

```js
export const meta = {
  name: 'orchestrate-backlog',
  description: 'Claim, implement, and review every ready-for-agent issue in parallel worktrees; one branch per issue, nothing merged',
  phases: [
    { title: 'Discover' },
    { title: 'Claim' },
    { title: 'Implement' },
    { title: 'Review & Fix', model: 'opus' },
  ],
}

const DISCOVERY_SCHEMA = {
  type: 'object',
  properties: {
    labelResolved: { type: 'boolean' },
    tasks: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          number: { type: 'integer' },
          title: { type: 'string' },
          body: { type: 'string' },
          estimatedComplexity: { type: 'string', enum: ['simple', 'standard'] },
          complexityReason: { type: 'string' },
        },
        required: ['number', 'title', 'body', 'estimatedComplexity'],
      },
    },
  },
  required: ['labelResolved', 'tasks'],
}

const CLAIM_SCHEMA = {
  type: 'object',
  properties: {
    ok: { type: 'boolean' },
    worktreePath: { type: 'string' },
    branch: { type: 'string' },
    baseSha: { type: 'string' },
    reason: { type: 'string' },
  },
  required: ['ok'],
}

const IMPLEMENT_SCHEMA = {
  type: 'object',
  properties: {
    worktreePath: { type: 'string' },
    branch: { type: 'string' },
    commitSha: { type: 'string' },
    handoff: { type: 'string' },
  },
  required: ['worktreePath', 'branch', 'commitSha', 'handoff'],
}

const REVIEW_SCHEMA = {
  type: 'object',
  properties: {
    committed: { type: 'boolean' },
    branch: { type: 'string' },
    commitSha: { type: 'string' },
    summary: { type: 'string' },
    discrepancies: { type: 'string' },
    worktreeRemoved: { type: 'boolean' },
    hitl: {
      type: 'object',
      properties: { triggered: { type: 'boolean' }, reason: { type: 'string' } },
      required: ['triggered'],
    },
  },
  required: ['committed', 'hitl'],
}

// Read args through this normalizer, never directly — see the note above. If you add a field,
// destructure it here too rather than reaching for `args.newField` mid-script.
let input
try {
  input = typeof args === 'string' ? JSON.parse(args) : args
} catch {
  throw new Error(`orchestrate-backlog got an args string it could not parse: ${String(args).slice(0, 200)}`)
}
const { repo, readyLabel, epicLabel, maxParallel, retryAfterMs } = input ?? {}
if (!repo || !readyLabel || !epicLabel) {
  throw new Error(
    `orchestrate-backlog requires repo, readyLabel, epicLabel — got ${JSON.stringify(args)} ` +
      `(typeof ${typeof args}). Fix the Workflow call's args and retry; never let the fleet run against ` +
      `undefined targets.`
  )
}
const LANES = Math.max(1, Math.min(8, maxParallel || 4))
const PROTECTED_BRANCHES = ['trunk', 'main', 'master']

// `agent()` returns null — it does NOT throw — when a subagent dies on a terminal API error after its
// own internal retries. A provider-side 529 overload takes out every in-flight agent at once, so this
// is a fleet-wide event, not a per-task fluke: re-spawning immediately just hands the overloaded server
// more of the same load. Back off, escalating, and give it room.
//
// The script cannot see the HTTP response, so a `Retry-After` header is invisible here. When the
// orchestrator DID observe one (in a failed run's error output), it passes the value as
// `retryAfterMs` and that becomes the floor. Otherwise the floor is a conservative 60s.
// Re-spawn with a byte-identical prompt/opts so a later `resumeFromRunId` still matches the cache.
const pause = typeof setTimeout === 'function' ? (ms) => new Promise((r) => setTimeout(r, ms)) : async () => {}
const BACKOFF_FLOOR_MS = Math.max(30_000, retryAfterMs || 60_000)
async function tryAgent(prompt, opts, attempts = 3) {
  for (let i = 1; i <= attempts; i++) {
    const result = await agent(prompt, opts)
    if (result) return result
    if (i === attempts) break
    const waitMs = BACKOFF_FLOOR_MS * 2 ** (i - 1)
    log(`${opts.label}: agent returned null (attempt ${i}/${attempts}) — backing off ${Math.round(waitMs / 1000)}s`)
    await pause(waitMs)
  }
  log(`${opts.label}: agent dead after ${attempts} attempts`)
  return null
}

// Bounded-concurrency pool: LANES lanes pull from one shared index, so at most LANES tasks are in
// flight and a lane refills the moment its task finishes. pipeline()/parallel() would instead run up
// to Workflow's own ~16 cap, which is what the 4-task limit exists to prevent.
async function pool(items, limit, run) {
  const results = new Array(items.length)
  let next = 0
  await Promise.all(
    Array.from({ length: Math.min(limit, items.length) }, async () => {
      for (let i = next++; i < items.length; i = next++) {
        try {
          results[i] = await run(items[i], i)
        } catch (err) {
          results[i] = { error: String(err) }
        }
      }
    })
  )
  return results
}

const done = []
const hitl = []
const failed = []
let round = 0
const MAX_ROUNDS = 20

while (round < MAX_ROUNDS) {
  round++
  phase('Discover')
  const discovery = await tryAgent(
    `Repo ${repo}. Read docs/agents/issue-tracker.md and docs/agents/triage-labels.md if they exist and ` +
      `follow their conventions exactly, including the real label strings. Find the frontier of ` +
      `AFK-ready work: open issues labelled "${readyLabel}", NOT labelled "${epicLabel}", with no ` +
      `assignee, and not blocked. An assignee means an earlier round already has it in flight — skip it, ` +
      `never re-claim it. Blocked means an open native issue dependency (gh api .../issues/<n> — check ` +
      `issue_dependencies_summary.blocked_by > 0) or an open issue referenced in a "Blocked by:" line in ` +
      `the body. If you cannot confirm the "${readyLabel}" label actually exists in this repo (gh label list), ` +
      `set labelResolved=false and return an empty tasks array — do NOT substitute a similar-looking label ` +
      `or fall back to listing all open issues. For each task, estimate complexity from its title/body alone: ` +
      `"standard" if it touches a core public API, unsafe code, an ADR-covered area, or a wide file footprint; ` +
      `"simple" otherwise. Do not use labels for that estimate. Return the frontier only.`,
    { schema: DISCOVERY_SCHEMA, label: `discover:round-${round}` }
  )

  if (!discovery) {
    log(`Round ${round}: discovery agent died even after retries — stopping the loop, keeping what already landed.`)
    break
  }

  if (!discovery.labelResolved) {
    throw new Error(
      `Discover could not confirm the "${readyLabel}" label exists in ${repo} — stopping before claiming anything.`
    )
  }

  if (!discovery.tasks.length) {
    log(round === 1 ? `No open, unblocked, unassigned "${readyLabel}" issues found.` : 'Frontier is dry — fleet done.')
    break
  }

  log(`Round ${round}: ${discovery.tasks.length} task(s), ${LANES} lane(s) in flight at a time.`)

  const results = await pool(discovery.tasks, LANES, async (task) => {
    const claim = await tryAgent(
      `Claim GitHub issue #${task.number} in ${repo}: gh issue edit ${task.number} --add-assignee @me. ` +
        `Then create an isolated worktree for it: git worktree add ../wt-issue-${task.number} -b ` +
        `afk/issue-${task.number} trunk — branched off the current tip of trunk, NOT off any other branch ` +
        `and NOT off another issue's branch. If trunk cannot be verified (git rev-parse --verify trunk), or ` +
        `the worktree path or branch name already exists, set ok=false with a reason and stop — do not invent ` +
        `an alternative name or reuse someone else's checkout. On success return ok=true with the worktree ` +
        `path, branch name, and the base SHA you branched from.`,
      { phase: 'Claim', schema: CLAIM_SCHEMA, label: `claim:${task.number}` }
    )
    // Null-check EVERY stage inside the lane. A null that escapes to the result handler below is
    // dereferenced outside this try/catch and takes down the whole workflow — see the incident note.
    if (!claim) throw new Error(`claim agent for #${task.number} died after retries — nothing claimed, no worktree created`)
    if (!claim.ok) throw new Error(`claim failed for #${task.number}: ${claim.reason ?? 'unknown reason'}`)
    if (PROTECTED_BRANCHES.includes(claim.branch)) {
      throw new Error(`claim for #${task.number} returned protected branch "${claim.branch}" — refusing to implement on it`)
    }

    const impl = await tryAgent(
      `In the worktree at ${claim.worktreePath} (branch ${claim.branch}), invoke /implement for ` +
        `issue #${task.number}: ${task.title}\n\n${task.body}\n\n` +
        `Work ONLY inside that worktree — never touch the primary checkout or another worktree. ` +
        `Do NOT invoke /code-review yourself; a separate reviewer agent handles that. Do NOT merge, ` +
        `rebase onto trunk, push, or open a PR — your branch is the deliverable and integration is a human ` +
        `step. When done, squash your work into one commit on ${claim.branch} following the repo's ` +
        `conventional-commit convention with an issue reference in the body, then invoke /handoff to write a ` +
        `handoff document (what you built, key decisions, where the diff is). Return the handoff text and the ` +
        `commit SHA.`,
      {
        phase: 'Implement',
        schema: IMPLEMENT_SCHEMA,
        label: `implement:${task.number}`,
        model: task.estimatedComplexity === 'simple' ? 'haiku' : 'sonnet',
      }
    )

    if (!impl) {
      throw new Error(
        `implement agent for #${task.number} died after retries — worktree ${claim.worktreePath} (branch ` +
          `${claim.branch}) is left in place, possibly dirty, and the issue stays assigned. Inspect it before rerunning.`
      )
    }

    const review = await tryAgent(
      `Reviewing issue #${task.number} in ${repo}. Use the SAME worktree the implementer used — do not ` +
        `create a new one: ${impl.worktreePath}, branch ${impl.branch}. You did not write this code. Read the ` +
        `handoff below, then run /code-review against the base commit ${claim.baseSha} (its Standards axis and ` +
        `its Spec axis — the Spec axis is where you compare the issue's acceptance criteria to what was ` +
        `actually delivered). Fix any findings yourself; the implementer gets no second pass. Report what the ` +
        `Spec axis found in the discrepancies field: specified-but-undelivered, delivered-but-unspecified, or ` +
        `diverging from intent, or "none".\n\n` +
        `The worktree may already hold uncommitted changes from an earlier reviewer that was killed mid-fix. ` +
        `Do not trust or blindly commit them: review the full diff against ${claim.baseSha} as if you were ` +
        `first to see it, and re-run the repo's validation gate before reporting success.\n\n` +
        `ABSOLUTE CONSTRAINTS — the whole point of this fleet:\n` +
        `- Do NOT merge, rebase onto trunk, fast-forward, cherry-pick, push, or open a PR. Nothing leaves ` +
        `${impl.branch}. If you feel the urge to integrate, that is the human's job, not yours.\n` +
        `- Do NOT close issue #${task.number} and do NOT unassign it. Comment on it with the branch name, ` +
        `the final commit SHA, and a short summary, then leave it open and assigned — that is the ` +
        `"awaiting human merge" state.\n` +
        `- If a fix would need an irreversible or high-blast-radius action, or a conflict you cannot resolve ` +
        `after a genuine attempt, STOP: change nothing further, report hitl.triggered=true with why, and leave ` +
        `the worktree in place for a human.\n` +
        `Otherwise: squash to one tidy commit on ${impl.branch}, verify git status --porcelain is empty in the ` +
        `worktree, then dispose of it — git worktree remove ${impl.worktreePath} && git worktree prune. Never ` +
        `pass --force: the refusal means uncommitted work is still there and is the safety check, not an ` +
        `obstacle. Keep the branch; only the worktree goes. Return committed=true with the branch, final SHA, ` +
        `and whether the worktree was removed.\n\nHandoff:\n${impl.handoff}`,
      { phase: 'Review & Fix', schema: REVIEW_SCHEMA, label: `review:${task.number}`, model: 'opus' }
    )

    if (!review) {
      throw new Error(
        `review agent for #${task.number} died after retries — the implementer's commit ${impl.commitSha} is ` +
          `safe on branch ${impl.branch}, but it is UNREVIEWED and worktree ${impl.worktreePath} is still live.`
      )
    }
    return { task, review }
  })

  results.forEach((r, i) => {
    const task = discovery.tasks[i]
    // Nothing in here may dereference an unchecked value: this handler runs outside the pool's
    // per-lane try/catch, so a TypeError here aborts the entire run — including work already committed.
    if (!r || r.error || !r.review) {
      failed.push({ number: task.number, title: task.title, reason: r?.error ?? 'agent returned null' })
      return
    }
    const { review } = r
    if (review.committed) {
      done.push({
        number: task.number,
        title: task.title,
        branch: review.branch ?? `afk/issue-${task.number}`,
        commitSha: review.commitSha,
        summary: review.summary,
        discrepancies: review.discrepancies,
        worktreeRemoved: review.worktreeRemoved !== false,
      })
    }
    if (review.hitl?.triggered) {
      hitl.push({ number: task.number, title: task.title, reason: review.hitl.reason })
    }
  })
}

if (round === MAX_ROUNDS) log(`Hit the ${MAX_ROUNDS}-round safety cap — some issues may remain unprocessed.`)

const leftBehind = done.filter((d) => !d.worktreeRemoved).map((d) => `#${d.number}`)
if (leftBehind.length) log(`Worktrees not confirmed removed for ${leftBehind.join(', ')} — sweep manually.`)

return { done, hitl, failed, roundsUsed: round, lanes: LANES }
```

## Resuming after a partial failure

A fleet-wide API failure kills every in-flight agent at once, so the usual shape of a bad run is
"all implementations committed, all reviews dead." The commits are the durable artifact and they
survive — recover, don't restart:

1. **Read `journal.jsonl` in the run's transcript dir first.** It records each agent's actual return
   value, so it tells you which stages really completed. Don't infer it from the error, and don't
   assume a cached result is non-empty.
2. **Check every worktree's state** — `git -C <wt> status --porcelain` and `git -C <wt> log --oneline -1`.
   A reviewer killed mid-fix leaves uncommitted changes; that partial work must be re-reviewed, never
   committed on trust. The review prompt already tells the resumed reviewer this.
3. **Edit the persisted script, then resume**: `Workflow({ scriptPath, resumeFromRunId })`. The
   longest unchanged prefix of `agent()` calls replays from cache, so successful implementations cost
   nothing on the second pass.
4. **Keep surviving prompts byte-identical.** The cache keys on `(prompt, opts)` in call order —
   reword a prompt and that call plus everything after it re-runs live. Adding a wrapper around
   `agent()` is safe; changing the string it receives is not.
5. **If the failure was rate/overload-driven, pass `retryAfterMs`** from whatever the provider asked
   for before resuming, and consider dropping `maxParallel` for the retry — four Opus reviewers
   starting within seconds of each other is itself part of the load.

## Notes on the design choices baked into this template

- **Nothing merges, so there is no shared branch and no ordering problem.** Each issue branches off
  the same trunk tip and stays there. That removes the whole class of races the epic-scoped sibling
  has to serialize around (two reviewers fast-forwarding the same feature branch), at the cost of
  handing the user N branches to integrate. Those branches may conflict with each other — the fleet
  cannot know, because no one ever tries the merge. Say so in the report rather than implying the
  branches are independently landable.
- **The constraint has to be stated in the reviewer prompt, not just the schema.** An Opus agent that
  has just fixed review findings will reach for `rebase`/`--ff-only` on its own, because that is what
  every other workflow in this repo asks for. The prohibition is spelled out as an explicit list, with
  the reason, because a bare "don't merge" reads as an oversight to be helpfully corrected.
- **A hand-rolled pool, not `pipeline()`.** `pipeline()` runs every item concurrently up to
  Workflow's own cap (`min(16, cores-2)`), which would put 16 issues in flight — the opposite of a
  4-task limit. The pool keeps per-item stage chaining (claim → implement → review runs sequentially
  *within* a lane, exactly as a pipeline would) while bounding how many lanes exist. `Promise.all` over
  lane functions is fine here; `Date.now`/`Math.random` are not available in Workflow scripts, so
  don't reach for jitter or timestamps.
- **Agent count scales with the backlog.** Three agents per issue plus one discovery agent per round:
  a 6-issue run is ~19 agents, above the default "keep it under 15" guideline. That's inherent to
  draining a backlog, not an accident — mention the expected count when the frontier is large, and
  let the user cut it with a smaller `maxParallel` or a narrower label if they'd rather not.
- **Errors are captured per lane, not fatal — but only inside the lane.** A lane's `try/catch` turns a
  failed claim or a dead agent into a `failed` entry so the other lanes keep working. The result
  handler that runs *after* the pool has no such protection, and that asymmetry caused the first real
  incident with this template: a provider-side **529 overload killed five agents simultaneously** (all
  four Opus reviewers plus one claim) at the point where all four implementations were already
  committed. Because `agent()` returns `null` instead of throwing, the null review sailed out of the
  lane, and `review.committed` in the `forEach` threw a `TypeError` that aborted the whole run — losing
  the report for work that was sitting safely on disk. Hence two rules, both load-bearing: **null-check
  every stage inside the lane**, and **let nothing in the result handler dereference an unchecked
  value**. A dead agent must degrade one lane, never the fleet.
- **Back off; don't hammer.** `tryAgent` re-spawns a dead call up to three times with escalating waits
  from a 60s floor. A 529 means the provider is overloaded, so an immediate retry adds to the very load
  that caused the failure — and since the whole fleet dies together, every lane would retry in lockstep.
  The script can't read a `Retry-After` header (it never sees the HTTP response), so when the
  orchestrator observed one, it passes `retryAfterMs` and that becomes the floor. Retries reuse a
  byte-identical prompt so `resumeFromRunId` still matches the cache.
- **Assignment is the concurrency guard; the label is the queue.** Discover excludes assigned issues,
  so an issue claimed in round 1 can't be re-claimed in round 2, and a human adding the label
  mid-run gets picked up by the next round. Dependency status is the ordering guard. Neither is a
  scheduler — both are filters at Discover time.
- **Worktree removed, branch kept, issue open.** These three are one decision: the commit is the
  durable artifact, so the checkout is disposable, the branch is not, and the issue can't be closed
  because nothing has landed. Removing a worktree requires it to be clean, which is exactly the check
  that no work is being discarded — hence the explicit ban on `--force`, which would silently throw
  away a reviewer's uncommitted fix.
- **Every external input fails loud.** `args` is validated before a single agent spawns; the label's
  existence is confirmed before Discover returns anything; the claim stage refuses a protected branch
  and refuses to reuse an existing worktree. This is not defensive boilerplate — the sibling skill's
  first run hit exactly this, with `args` arriving JSON-stringified and six review agents responding
  to an unresolvable branch name by defaulting to `trunk`. A malformed input must stop the fleet, never
  be treated as "good enough, proceed".
