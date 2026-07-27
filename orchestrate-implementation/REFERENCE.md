# Workflow script template

Adapt this before calling `Workflow`. It's written against the GitHub-native-sub-issues convention
described in `docs/agents/issue-tracker.md`; if the target repo's tracker file describes something
else (checklist fallback, a different CLI), change the prose inside the `agent()` prompts — the
control flow (discover → pipeline → loop → discrepancy check) stays the same.

Pass `args: { epic: <number>, repo: "<owner>/<repo>", featureBranch: "feature/epic-<number>" }` on
the `Workflow` call. Create `featureBranch` off `trunk` yourself (a local branch, no push) before
invoking, if it doesn't already exist. Add `retryAfterMs: <ms>` when a previous run surfaced a
`Retry-After` header — it becomes the backoff floor.

> **Known issue — `args` may arrive JSON-stringified.** Observed repeatedly across separate projects
> and invocations (with and without `resumeFromRunId`): the object passed as `args` reaches the script
> as a *string* holding its JSON. `args` is the only `Workflow` input with no declared type, so nothing
> enforces object-ness. The script normalizes for this (parse-if-string, below) and hard-fails if the
> result is still incomplete — but if you see it recur, **skip `args` entirely and inline the config as
> literals** in the script instead:
>
> ```js
> const cfg = { epic: 42, repo: 'owner/repo', featureBranch: 'feature/epic-42' }
> ```
>
> then destructure from `cfg` rather than `input`. Workflow persists a script per invocation and you
> iterate via `scriptPath` anyway, so a per-epic literal costs nothing in reusability. Keep the guard
> when you do this, but understand what it becomes: checked against literals it can no longer catch a
> transport failure — only a typo, which is still worth catching for the protected-branch case.

```js
export const meta = {
  name: 'orchestrate-implementation',
  description: 'Claim, implement, review, and land open sub-tasks of an epic in parallel, in isolated worktrees',
  phases: [
    { title: 'Discover' },
    { title: 'Claim' },
    { title: 'Implement' },
    { title: 'Review & Fix', model: 'opus' },
    { title: 'Discrepancy check' },
  ],
}

const DISCOVERY_SCHEMA = {
  type: 'object',
  properties: {
    epicFound: { type: 'boolean' },
    subtasks: {
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
  required: ['epicFound', 'subtasks'],
}

const CLAIM_SCHEMA = {
  type: 'object',
  properties: {
    ok: { type: 'boolean' },
    worktreePath: { type: 'string' },
    branch: { type: 'string' },
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
    landed: { type: 'boolean' },
    commitSha: { type: 'string' },
    summary: { type: 'string' },
    hitl: {
      type: 'object',
      properties: {
        triggered: { type: 'boolean' },
        reason: { type: 'string' },
      },
      required: ['triggered'],
    },
  },
  required: ['landed', 'hitl'],
}

// `args` can arrive JSON-stringified instead of as an object (observed in practice — `args` is the
// one Workflow input with no declared type, so nothing enforces object-ness). On a string, EVERY
// property read yields undefined, which then interpolates into prompts as the literal text
// "undefined" — silent, not a TypeError. Normalize first, then fail fast on whatever's still missing.
let input
try {
  input = typeof args === 'string' ? JSON.parse(args) : args
} catch {
  throw new Error(
    `orchestrate-implementation got an args string it could not parse: ${String(args).slice(0, 200)}`
  )
}
const { epic, repo, featureBranch, retryAfterMs } = input ?? {}
const PROTECTED_BRANCHES = ['trunk', 'main', 'master']
if (!epic || !repo || !featureBranch) {
  throw new Error(
    `orchestrate-implementation requires epic, repo, featureBranch — got ${JSON.stringify(args)} ` +
      `(typeof ${typeof args}). Fix the Workflow call's args and retry; never let the fleet run against ` +
      `undefined targets.`
  )
}
if (PROTECTED_BRANCHES.includes(featureBranch)) {
  throw new Error(
    `featureBranch ("${featureBranch}") is a protected branch — sub-tasks must never land directly on it.`
  )
}

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

const landed = []
const hitl = []
const failed = []
let round = 0
const MAX_ROUNDS = 20

while (round < MAX_ROUNDS) {
  round++
  phase('Discover')
  const discovery = await tryAgent(
    `Repo ${repo}. First confirm epic #${epic} exists and is open (gh issue view ${epic}). If it ` +
      `does not exist, or you cannot confirm it, set epicFound=false and return an empty subtasks array — do ` +
      `NOT substitute a similar-looking issue from a different epic. If confirmed, read ` +
      `docs/agents/issue-tracker.md if it exists and follow its child-discovery and blocking conventions ` +
      `(e.g. its "frontier query" recipe) exactly. If it doesn't exist, use plain gh: children of epic ` +
      `#${epic} are issues linked via gh's native sub-issue API, or failing that issues whose body ` +
      `contains "Part of #${epic}"; an issue is blocked if it has an open native issue dependency (gh api ` +
      `.../issues/<n> — check issue_dependencies_summary.blocked_by > 0) or an open issue referenced in a ` +
      `"Blocked by:" line in its body. Return only children that are open, unblocked, and unassigned (an ` +
      `assignee means another round's pipeline already has it in flight — skip it, don't re-claim it). For ` +
      `each, estimate complexity from its title/body alone — "standard" if it touches a core public API, ` +
      `unsafe code, an ADR-covered area, or a wide file footprint; "simple" otherwise. Do not rely on labels ` +
      `for this estimate. Return the frontier only, not the whole epic.`,
    { schema: DISCOVERY_SCHEMA, label: `discover:round-${round}` }
  )

  if (!discovery) {
    log(`Round ${round}: discovery agent died even after retries — stopping the loop, keeping what already landed.`)
    break
  }

  if (!discovery.epicFound) {
    throw new Error(`Discover could not confirm epic #${epic} exists in ${repo} — stopping before claiming anything.`)
  }

  if (!discovery.subtasks.length) {
    log(round === 1 ? 'No open, unblocked sub-tasks found.' : 'No more eligible sub-tasks — fleet is dry.')
    break
  }

  const results = await pipeline(
    discovery.subtasks,
    (task) =>
      tryAgent(
        `Claim GitHub issue #${task.number} in ${repo}: gh issue edit ${task.number} --add-assignee @me. ` +
          `Then confirm branch "${featureBranch}" actually exists locally (git rev-parse --verify ` +
          `${featureBranch}). If it does not exist, set ok=false with a reason and stop there — do NOT ` +
          `fall back to trunk, main, master, or any invented branch name, no matter how plausible it looks. ` +
          `If it exists, create a git worktree for this sub-task at ../wt-subtask-${task.number} on new branch ` +
          `subtask-${task.number} based on it, set ok=true, and return the worktree path and branch name.`,
        { phase: 'Claim', schema: CLAIM_SCHEMA, label: `claim:${task.number}` }
      ),
    (claim, task) => {
      // Null-check every stage: `agent()` returns null on a dead subagent, and `claim.ok` on a null
      // would be a TypeError. A throw here is contained — pipeline() drops the item to null and skips
      // its remaining stages — so this degrades one sub-task, not the fleet.
      if (!claim) {
        throw new Error(`claim agent for #${task.number} died after retries — nothing claimed, no worktree created`)
      }
      if (!claim.ok) {
        throw new Error(`claim failed for #${task.number}: ${claim.reason ?? 'unknown reason'} — skipping implement/review`)
      }
      return tryAgent(
        `In the worktree at ${claim.worktreePath} (branch ${claim.branch}), invoke /implement for ` +
          `sub-task #${task.number}: ${task.title}\n\n${task.body}\n\n` +
          `Do NOT invoke /code-review yourself — a separate reviewer agent handles that. ` +
          `When done, squash your work into one commit on the branch, then invoke /handoff to write a ` +
          `handoff document (what you built, key decisions, where the diff is). Return the handoff text ` +
          `and the commit SHA.`,
        {
          phase: 'Implement',
          schema: IMPLEMENT_SCHEMA,
          label: `implement:${task.number}`,
          model: task.estimatedComplexity === 'simple' ? 'haiku' : 'sonnet',
        }
      )
    },
    (impl, task) => {
      if (!impl) {
        throw new Error(
          `implement agent for #${task.number} died after retries — its worktree is left in place, possibly ` +
            `dirty, and the issue stays assigned. Inspect it before rerunning.`
        )
      }
      return tryAgent(
        `Reviewing sub-task #${task.number}. Use the SAME worktree the implementer used — do not create ` +
          `a new one: ${impl.worktreePath}, branch ${impl.branch}. You did not write this code; read the ` +
          `handoff below, then run /code-review against ${featureBranch}. Fix any findings yourself — ` +
          `the implementer does not get a second pass. The ONLY valid merge target is the exact branch ` +
          `"${featureBranch}" — it is never trunk/main/master and you must never substitute one of those ` +
          `or invent a different name, even as a fallback. Before merging, re-verify with git rev-parse ` +
          `--verify ${featureBranch} that it still exists and is what you think it is. If it's missing, ` +
          `looks wrong, or a fix would require an irreversible/high-blast-radius action, or a rebase/merge ` +
          `conflict against it can't be resolved after a genuine attempt, STOP without merging anything ` +
          `anywhere and report hitl.triggered=true with why. Otherwise: squash to one tidy commit, rebase ` +
          `onto ${featureBranch}, merge --ff-only into ${featureBranch}. Then comment on issue ` +
          `#${task.number} with the commit link and a short summary, close it, and remove the worktree (git ` +
          `worktree remove ${impl.worktreePath} --force). Never push or open a PR.\n\nHandoff:\n${impl.handoff}`,
        { phase: 'Review & Fix', schema: REVIEW_SCHEMA, label: `review:${task.number}`, model: 'opus' }
      )
    }
  )

  // Index into `discovery.subtasks` with the UNFILTERED index — `pipeline()` returns one slot per
  // input item, nulling the dropped ones. Filtering first would shift every later result onto the
  // wrong sub-task and report commits against issue numbers that never produced them. Nothing in
  // here may dereference an unchecked value: this handler runs outside `pipeline()`'s per-item error
  // containment, so a TypeError aborts the entire run — including sub-tasks already merged.
  results.forEach((r, i) => {
    const task = discovery.subtasks[i]
    if (!r) {
      failed.push({ number: task.number, title: task.title, reason: 'a stage threw or its agent died — see the log' })
      return
    }
    if (r.landed) landed.push({ number: task.number, title: task.title, summary: r.summary, commitSha: r.commitSha })
    if (r.hitl?.triggered) hitl.push({ number: task.number, title: task.title, reason: r.hitl.reason })
  })
}

if (round === MAX_ROUNDS) log(`Hit the ${MAX_ROUNDS}-round safety cap — some sub-tasks may remain undiscovered.`)

phase('Discrepancy check')
const discrepancies = await tryAgent(
  `Compare epic #${epic}'s spec (title, body, acceptance criteria) in ${repo} against what ` +
    `actually landed on ${featureBranch} (git log/diff against its merge-base with trunk, plus the ` +
    `closed sub-issues and their summary comments). Report anything specified but not delivered, delivered ` +
    `but not specified, or diverging from the spec's intent.`,
  {
    schema: { type: 'object', properties: { discrepancies: { type: 'string' } }, required: ['discrepancies'] },
    label: 'discrepancy-check',
  }
)

// This is the last call in the run, so a bare `discrepancies.discrepancies` would throw away the
// entire report over one dead agent — after every sub-task had already merged. Degrade instead.
return {
  landed,
  hitl,
  failed,
  discrepancies: discrepancies?.discrepancies ?? 'UNKNOWN — the discrepancy-check agent died; re-run it manually.',
  roundsUsed: round,
}
```

## Resuming after a partial failure

A fleet-wide API failure kills every in-flight agent at once, so the usual shape of a bad run is
"all implementations committed, all reviews dead." Those commits are durable and survive on their
sub-task branches — recover, don't restart:

1. **Read `journal.jsonl` in the run's transcript dir first.** It records each agent's actual return
   value, so it tells you which stages really completed. Don't infer it from the error, and don't
   assume a cached result is non-empty.
2. **Check every worktree's state** — `git -C <wt> status --porcelain` and `git -C <wt> log --oneline -1`.
   A reviewer killed mid-fix leaves uncommitted changes; that partial work must be re-reviewed against
   the base, never committed on trust.
3. **Edit the persisted script, then resume**: `Workflow({ scriptPath, resumeFromRunId })`. The longest
   unchanged prefix of `agent()` calls replays from cache, so successful implementations cost nothing.
4. **Keep surviving prompts byte-identical.** The cache keys on `(prompt, opts)` in call order —
   reword a prompt and that call plus everything after it re-runs live. Wrapping `agent()` is safe;
   changing the string it receives is not.
5. **If the failure was rate/overload-driven, pass `retryAfterMs`** from whatever the provider asked
   for, and consider fewer sub-tasks per round — several Opus reviewers starting within seconds of
   each other is itself part of the load.

## Notes on the design choices baked into this template

- **Worktree reuse is scoped to one sub-task's lifecycle** (implement → review → fix), not shared
  across sub-tasks — two sub-tasks can run concurrently in the pipeline, and a single `git worktree`
  can only have one branch checked out at a time. This is why the claim stage creates the worktree
  explicitly with `git worktree add` rather than via `agent()`'s `opts.isolation: 'worktree'` (that
  option gives each *call* its own fresh worktree, which would hand the reviewer a different checkout
  than the implementer used).
- **`pipeline()` over `parallel()`** for claim/implement/review: sub-tasks don't need to be
  synchronized with each other at any stage, so the slowest sub-task doesn't block the rest — a
  `parallel()` barrier here would waste wall-clock for no benefit (see the Workflow tool's own
  guidance on when a barrier is/isn't justified).
- **The outer `while` loop re-runs discovery** because landing one sub-task can unblock another
  (a dependency edge closes). The `MAX_ROUNDS` cap is a backstop, not a design target — it's logged
  when hit rather than silently truncating, since the caller needs to know coverage may be partial.
- **Blocking is a filter at Discover time, not a scheduler.** The Discover prompt must resolve
  children-of-epic and blocked-by exactly the way `docs/agents/issue-tracker.md` documents (or the
  plain-`gh` fallback) — don't leave it to the sub-agent to guess the convention. The same Discover
  call also excludes already-*assigned* issues, which is what keeps a sub-task claimed in an earlier,
  still-in-flight round from being re-claimed by a later round's pipeline — assignment doubles as the
  concurrency guard, dependency status doubles as the ordering guard.
- **Model tiers**: `haiku` for the implementer only when the sub-task's own text reads as narrow and
  low-risk; `sonnet` otherwise. The reviewer is always `opus`, per the brief, regardless of the
  sub-task's tier — review quality doesn't scale down just because the change was simple.
- **Every external input fails loud, not silent.** `args` is checked before a single agent spawns;
  the epic's existence is checked before Discover returns anything; the feature branch's existence is
  checked before Claim creates a worktree, and re-checked before Review merges. This is deliberate,
  not defensive boilerplate: a first run of this template hit exactly the failure mode these checks
  close — `args` arrived **JSON-stringified rather than as an object** (confirmed from the persisted
  run record: `args` was a `str` holding `'{"epic": 109, ...}'`). The keys were all correct; the
  container wasn't. On a string primitive every property read returns `undefined` rather than
  throwing, so every prompt silently interpolated the literal text `"undefined"`, and six review
  agents responded to an unresolvable merge target by defaulting
  to `trunk` — exactly the branch the skill's rules forbid touching — while a seventh, faced with the
  same broken input, correctly refused trunk but landed on an orphaned branch instead. A missing or
  malformed `args` field, or a merge target that isn't verifiably the real `featureBranch`, must stop
  the fleet (or that one sub-task) outright — it must never be treated as "good enough, proceed."
- **A dead agent must degrade one sub-task, never the fleet.** `agent()` returns `null` rather than
  throwing when a subagent dies on a terminal API error, and a provider-side **529 overload kills every
  in-flight agent simultaneously** — observed on the sibling skill's first real run, where four
  implementations were already committed and all four Opus reviewers died at once. Inside `pipeline()` a
  throw is contained (the item drops to `null` and skips its remaining stages), which is why each stage
  null-checks its input and throws a descriptive error. What is *not* contained is anything after the
  pipeline: on that run a null review reached `review.committed` in the result handler and the resulting
  `TypeError` aborted a workflow whose work was sitting safely on disk. So: null-check every stage, and
  let nothing outside `pipeline()` — result handler or final discrepancy read — dereference an unchecked
  value.
- **Back off; don't hammer.** `tryAgent` re-spawns a dead call up to three times with escalating waits
  from a 60s floor. A 529 means the provider is overloaded, so an immediate retry adds to the very load
  that caused the failure — and since the whole fleet dies together, every lane would retry in lockstep.
  The script can't read a `Retry-After` header (it never sees the HTTP response), so when the
  orchestrator observed one it passes `retryAfterMs` and that becomes the floor. Retries reuse a
  byte-identical prompt so `resumeFromRunId` still matches the cache.
- **Index results with the unfiltered index.** `pipeline()` returns one slot per input item, nulling
  the dropped ones, so `results.filter(Boolean).forEach((r, i) => discovery.subtasks[i])` shifts every
  result after the first failure onto the wrong sub-task — reporting commits and HITL reasons against
  issue numbers that never produced them. Filter *after* pairing, never before indexing.
- **Read `args` through the normalizer, never directly.** The script destructures `epic`/`repo`/
  `featureBranch` once at the top and interpolates those bindings; `args` itself is referenced only
  in the error message. `args` is the sole `Workflow` input with no declared type in its schema, so
  nothing upstream guarantees it arrives as an object — a stringified value is a live possibility on
  any run, not a one-off. If you add a new field, destructure it up there too rather than reaching
  for `args.newField` mid-script, or you reintroduce the silent-`undefined` hole for that one field.
