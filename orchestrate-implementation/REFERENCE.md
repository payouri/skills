# Workflow script templates

Two scripts, run in that order with act 2 — `/adversarial-code-review` — between them:

- **Workflow A — build**: discover → claim → implement → serial integrate, looped until dry.
- **Workflow B — repair**: triage → fix in worktrees → serial integrate → discrepancy check.

Both are written against the GitHub-native-sub-issues convention described in
`docs/agents/issue-tracker.md`; if the target repo's tracker file describes something else (checklist
fallback, a different CLI), change the prose inside the `agent()` prompts — the control flow stays
the same.

## Shared preamble

Both scripts open with the same normalizer, guards, and `tryAgent`. It is reproduced in each script
below rather than referenced, because a Workflow script must be self-contained.

`args` for both:

```json
{
  "epic": 42,
  "repo": "owner/repo",
  "repoPath": "/abs/path/to/primary/checkout",
  "featureBranch": "feature/epic-42",
  "baseBranch": "trunk",
  "baseSha": "9f2c1ab",
  "retryAfterMs": 60000
}
```

`repoPath` is the primary checkout where `featureBranch` is checked out — the only place the
integrator runs `git merge`. Workflow B additionally takes `findings`. Add `retryAfterMs` when a
previous run surfaced a `Retry-After` header; it becomes the backoff floor.

**`baseBranch` is a name; `baseSha` is the fixed point.** The base branch is whatever this repo
integrates into — `trunk`, `main`, a release branch, a parent feature branch — resolved in step 0, and
used only to create the feature branch and to name things in prompts. Every *measurement* uses
`baseSha`, the SHA of `featureBranch` captured before the first sub-task landed. Two different failures
are closed by that split: a hardcoded `trunk` is simply wrong in a repo that calls it something else,
and *any* branch name as a fixed point silently re-scopes the review if the base moves mid-run —
someone lands on `main` while act 1 is building, and a `main...HEAD` diff quietly starts including or
excluding work the epic never touched. A SHA cannot drift.

> **Known issue — `args` may arrive JSON-stringified.** Observed repeatedly across separate projects
> and invocations (with and without `resumeFromRunId`): the object passed as `args` reaches the script
> as a _string_ holding its JSON. `args` is the only `Workflow` input with no declared type, so nothing
> enforces object-ness. The scripts normalize for this (parse-if-string, below) and hard-fail if the
> result is still incomplete — but if you see it recur, **skip `args` entirely and inline the config as
> literals** in the script instead:
>
> ```js
> const cfg = {
>   epic: 42, repo: 'owner/repo', repoPath: '/abs/path',
>   featureBranch: 'feature/epic-42', baseBranch: 'trunk', baseSha: '9f2c1ab',
> }
> ```
>
> then destructure from `cfg` rather than `input`. Workflow persists a script per invocation and you
> iterate via `scriptPath` anyway, so a per-epic literal costs nothing in reusability. Keep the guard
> when you do this, but understand what it becomes: checked against literals it can no longer catch a
> transport failure — only a typo, which is still worth catching for the protected-branch case.

---

## Workflow A — build

```js
export const meta = {
  name: 'orchestrate-implementation-build',
  description: 'Claim, implement, and land open sub-tasks of an epic in parallel worktrees — no review',
  phases: [
    { title: 'Discover' },
    { title: 'Claim' },
    { title: 'Implement' },
    { title: 'Integrate' },
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
    summary: { type: 'string', description: 'what was built and the key decisions — becomes the issue-close comment' },
  },
  required: ['worktreePath', 'branch', 'commitSha', 'summary'],
}

// The integrator handles every branch in one call, so it returns one row per sub-task it was given.
const INTEGRATE_SCHEMA = {
  type: 'object',
  properties: {
    results: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          number: { type: 'integer' },
          landed: { type: 'boolean' },
          commitSha: { type: 'string' },
          hitl: {
            type: 'object',
            properties: { triggered: { type: 'boolean' }, reason: { type: 'string' } },
            required: ['triggered'],
          },
        },
        required: ['number', 'landed', 'hitl'],
      },
    },
  },
  required: ['results'],
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
const { epic, repo, repoPath, featureBranch, baseBranch, baseSha, retryAfterMs } = input ?? {}
// The base branch may be called anything, so the protected set is "the usual names, plus whatever
// this repo actually integrates into". featureBranch must be neither.
const PROTECTED_BRANCHES = ['trunk', 'main', 'master', baseBranch].filter(Boolean)
if (!epic || !repo || !repoPath || !featureBranch || !baseSha) {
  throw new Error(
    `orchestrate-implementation requires epic, repo, repoPath, featureBranch, baseSha — got ` +
      `${JSON.stringify(args)} (typeof ${typeof args}). Fix the Workflow call's args and retry; never let ` +
      `the fleet run against undefined targets.`
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

  // claim → implement only. No review stage: the whole branch is red-teamed once, later, by act 2.
  const built = await pipeline(
    discovery.subtasks,
    (task) =>
      tryAgent(
        `Claim GitHub issue #${task.number} in ${repo}: gh issue edit ${task.number} --add-assignee @me. ` +
          `Then confirm branch "${featureBranch}" actually exists locally (git -C ${repoPath} rev-parse ` +
          `--verify ${featureBranch}). If it does not exist, set ok=false with a reason and stop there — do ` +
          `NOT fall back to ${PROTECTED_BRANCHES.join('/')}, or any invented branch name, no matter how ` +
          `plausible it looks. If it exists, create a git worktree for this sub-task at ${repoPath}/../wt-subtask-${task.number} ` +
          `on new branch subtask-${task.number} based on ${featureBranch}, set ok=true, and return the ` +
          `worktree path and branch name.`,
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
        throw new Error(`claim failed for #${task.number}: ${claim.reason ?? 'unknown reason'} — skipping implement`)
      }
      return tryAgent(
        `In the worktree at ${claim.worktreePath} (branch ${claim.branch}), invoke /implement for ` +
          `sub-task #${task.number}: ${task.title}\n\n${task.body}\n\n` +
          `You implement and nothing else. Do NOT invoke /code-review or /adversarial-code-review, and do ` +
          `NOT review your own diff — the entire feature branch is red-teamed once, later, by a separate ` +
          `fleet that did not write any of it. Skip /implement's own "now run /code-review" step. ` +
          `Do NOT merge anything, do NOT touch ${featureBranch}, and do NOT close or comment on the issue — ` +
          `a serial integrator does all of that after you finish. ` +
          `When done, squash your work into ONE commit on ${claim.branch} and return its SHA plus a short ` +
          `summary of what you built and the decisions behind it.`,
        {
          phase: 'Implement',
          schema: IMPLEMENT_SCHEMA,
          label: `implement:${task.number}`,
          model: task.estimatedComplexity === 'simple' ? 'haiku' : 'sonnet',
        }
      )
    }
  )

  // Barrier is REQUIRED here, unlike the pipeline above: integration is serial by design, so the
  // integrator needs the full set of finished branches before it can order them.
  // Pair with the UNFILTERED index — `pipeline()` returns one slot per input item, nulling dropped
  // ones, so filtering before pairing shifts every later result onto the wrong sub-task.
  const ready = []
  built.forEach((impl, i) => {
    const task = discovery.subtasks[i]
    if (!impl) {
      failed.push({ number: task.number, title: task.title, reason: 'claim or implement threw / its agent died — see the log' })
      return
    }
    ready.push({ ...task, ...impl })
  })

  if (!ready.length) {
    log(`Round ${round}: nothing implemented successfully — skipping integration.`)
    continue
  }

  phase('Integrate')
  const integration = await tryAgent(
    `You are the sole integrator for this round. Work in the primary checkout ${repoPath}, where ` +
      `${featureBranch} is checked out. Handle the sub-task branches below ONE AT A TIME, in the order ` +
      `listed — never in parallel, never in a worktree.\n\n` +
      ready
        .map((r) => `- #${r.number} "${r.title}" · branch ${r.branch} · commit ${r.commitSha} · worktree ${r.worktreePath}\n  summary: ${r.summary}`)
        .join('\n') +
      `\n\nFor each, in order:\n` +
      `1. Re-verify the merge target: git -C ${repoPath} rev-parse --verify ${featureBranch}. The ONLY ` +
      `valid target is the exact branch "${featureBranch}" — it is never ${PROTECTED_BRANCHES.join('/')} ` +
      `and you must never substitute one of those or invent a different name, even as a fallback. If it is missing or ` +
      `looks wrong, STOP the whole round and report hitl.triggered=true on every remaining sub-task.\n` +
      `2. Rebase the sub-task branch onto ${featureBranch}, squashed to one tidy commit.\n` +
      `3. If the rebase conflicts AT ALL: git rebase --abort, leave the branch and worktree in place, ` +
      `leave the issue open and assigned, set landed=false and hitl.triggered=true with the conflicting ` +
      `paths as the reason, and move on to the next sub-task. Do NOT resolve the conflict — a guessed ` +
      `resolution corrupts the branch silently. Do NOT skip ahead to a later branch to "unblock" it.\n` +
      `4. On a clean rebase: git -C ${repoPath} merge --ff-only <branch>. Then comment on issue ` +
      `#<number> with the commit link and the summary above, close it, and remove the worktree ` +
      `(git worktree remove <path> --force). Set landed=true with the resulting SHA.\n\n` +
      `Never push and never open a PR. Return one row per sub-task listed above, in the same order.`,
    { phase: 'Integrate', schema: INTEGRATE_SCHEMA, label: `integrate:round-${round}`, model: 'sonnet' }
  )

  // Nothing out here may dereference an unchecked value: this runs outside `pipeline()`'s per-item
  // error containment, so a TypeError aborts the whole run — including work already merged.
  const rows = integration?.results ?? []
  ready.forEach((r) => {
    const row = rows.find((x) => x?.number === r.number)
    if (!row) {
      failed.push({
        number: r.number,
        title: r.title,
        reason: `integrator returned no row for #${r.number} — its commit ${r.commitSha} is safe on branch ${r.branch}; land it by hand`,
      })
      return
    }
    if (row.landed) {
      landed.push({
        number: r.number,
        title: r.title,
        summary: r.summary,
        commitSha: row.commitSha,
        model: r.estimatedComplexity === 'simple' ? 'haiku' : 'sonnet',
      })
    }
    if (row.hitl?.triggered) hitl.push({ number: r.number, title: r.title, reason: row.hitl.reason })
  })
}

if (round === MAX_ROUNDS) log(`Hit the ${MAX_ROUNDS}-round safety cap — some sub-tasks may remain undiscovered.`)

return { landed, hitl, failed, roundsUsed: round }
```

---

## Act 2 — the review, between the workflows

Not a script. Invoke `/adversarial-code-review` and follow its SKILL.md, with:

- **fixed point** `baseSha` — the SHA pinned in step 0, never a branch name — verified from `repoPath`
  where `featureBranch` is checked out. Its §1 check is `git rev-parse <baseSha>` and
  `git diff <baseSha>...HEAD --stat`;
- **all five lenses** passed explicitly, so its lens-selection `AskUserQuestion` doesn't fire;
- the **epic issue body** as the Spec lens's rulebook.

Keep its full output. Workflow B consumes the `CONFIRMED`/`PLAUSIBLE` findings; the `REFUTED` and
`UNJUDGED` ones go straight to the final report.

---

## Workflow B — repair

`args` is the shared preamble's object plus `findings`, each shaped as the review returned it:

```json
{
  "findings": [
    {
      "lens": "Bugs",
      "id": "off-by-one-cursor",
      "claim": "…",
      "location": "src/cursor.rs:88",
      "hunk": "…",
      "failureScenario": "…",
      "severity": "blocker",
      "fix": "…",
      "verdict": "CONFIRMED",
      "verdictReason": "…"
    }
  ]
}
```

```js
export const meta = {
  name: 'orchestrate-implementation-repair',
  description: 'Fix every non-refuted review finding at a fix-complexity-tiered model, land them, then check the epic spec',
  phases: [
    { title: 'Triage' },
    { title: 'Fix' },
    { title: 'Integrate fixes' },
    { title: 'Discrepancy check' },
  ],
}

const TRIAGE_SCHEMA = {
  type: 'object',
  properties: {
    groups: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          key: { type: 'string', description: 'echo back the group key you were given' },
          tier: { type: 'string', enum: ['trivial', 'moderate', 'intricate'] },
          reason: { type: 'string' },
        },
        required: ['key', 'tier'],
      },
    },
  },
  required: ['groups'],
}

const FIX_SCHEMA = {
  type: 'object',
  properties: {
    branch: { type: 'string' },
    worktreePath: { type: 'string' },
    commitSha: { type: 'string' },
    fixed: { type: 'array', items: { type: 'string' }, description: 'finding ids actually fixed' },
    skipped: {
      type: 'array',
      items: {
        type: 'object',
        properties: { id: { type: 'string' }, reason: { type: 'string' } },
        required: ['id', 'reason'],
      },
    },
    hitl: {
      type: 'object',
      properties: { triggered: { type: 'boolean' }, reason: { type: 'string' } },
      required: ['triggered'],
    },
  },
  required: ['branch', 'fixed', 'skipped', 'hitl'],
}

const INTEGRATE_SCHEMA = {
  type: 'object',
  properties: {
    results: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          key: { type: 'string' },
          landed: { type: 'boolean' },
          commitSha: { type: 'string' },
          reason: { type: 'string', description: 'why it did not land, when landed=false without a HITL' },
          hitl: {
            type: 'object',
            properties: { triggered: { type: 'boolean' }, reason: { type: 'string' } },
            required: ['triggered'],
          },
        },
        required: ['key', 'landed', 'hitl'],
      },
    },
  },
  required: ['results'],
}

let input
try {
  input = typeof args === 'string' ? JSON.parse(args) : args
} catch {
  throw new Error(`orchestrate-implementation repair got an args string it could not parse: ${String(args).slice(0, 200)}`)
}
const { epic, repo, repoPath, featureBranch, baseBranch, baseSha, findings, retryAfterMs } = input ?? {}
const PROTECTED_BRANCHES = ['trunk', 'main', 'master', baseBranch].filter(Boolean)
if (!epic || !repo || !repoPath || !featureBranch || !baseSha || !Array.isArray(findings)) {
  throw new Error(
    `repair requires epic, repo, repoPath, featureBranch, baseSha, findings[] — got ${JSON.stringify(args)} (typeof ${typeof args}).`
  )
}
if (PROTECTED_BRANCHES.includes(featureBranch)) {
  throw new Error(`featureBranch ("${featureBranch}") is a protected branch — fixes must never land directly on it.`)
}

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

// Only CONFIRMED and PLAUSIBLE are repaired. REFUTED is dead. UNJUDGED means the refuter died before
// verdicting it — nothing has vetted the claim, so it is reported to the human, never handed to a
// fixer who would take it on faith.
const repairable = findings.filter((f) => f?.verdict === 'CONFIRMED' || f?.verdict === 'PLAUSIBLE')
const unjudged = findings.filter((f) => f?.verdict === 'UNJUDGED')
if (unjudged.length) log(`${unjudged.length} UNJUDGED finding(s) reported, not fixed — their refuter never verdicted them.`)

// Group by file. Two fixers editing the same file in different worktrees produce two commits that
// conflict at integration time, so the file is the unit of concurrency, not the finding.
const byFile = new Map()
for (const f of repairable) {
  const file = String(f.location ?? 'unknown').split(':')[0]
  if (!byFile.has(file)) byFile.set(file, [])
  byFile.get(file).push(f)
}
const groups = [...byFile.entries()].map(([file, items], i) => ({
  key: `g${i}`,
  file,
  slug: `fix-${i}-${file.replace(/[^a-zA-Z0-9]+/g, '-').slice(-40).replace(/^-+/, '')}`,
  findings: items,
}))

if (!groups.length) {
  log('No CONFIRMED or PLAUSIBLE findings — skipping straight to the discrepancy check.')
}

const renderFindings = (items) =>
  items
    .map(
      (f) =>
        `- id: ${f.id} · lens: ${f.lens} · verdict: ${f.verdict} · severity: ${f.severity}\n` +
        `  claim: ${f.claim}\n  at: ${f.location}\n  fails when: ${f.failureScenario}\n` +
        `  suggested fix: ${f.fix}\n  refuter said: ${f.verdictReason ?? ''}`
    )
    .join('\n')

let triage = { groups: [] }
if (groups.length) {
  phase('Triage')
  triage =
    (await tryAgent(
      `Assign a model tier to each group of review findings below, by how hard the FIX is — not by how ` +
        `severe the defect is. A blocker fixed by flipping one comparison is "trivial". A minor smell whose ` +
        `fix reshapes an interface, touches concurrency or unsafe code, or requires judgement about intent is ` +
        `"intricate". Everything else is "moderate". Read the cited code in ${repoPath} (branch ` +
        `${featureBranch}) before deciding — a suggested fix that looks small in prose is often not.\n\n` +
        groups.map((g) => `## group ${g.key} — file ${g.file}\n${renderFindings(g.findings)}`).join('\n\n') +
        `\n\nReturn one row per group key above, inventing no keys.`,
      { phase: 'Triage', schema: TRIAGE_SCHEMA, label: 'triage-fix-complexity', model: 'sonnet' }
    )) ?? { groups: [] }
}

// A dead triage agent must not silently downgrade every fix to the cheapest model. Default to the
// most capable tier instead: over-spending on a fix is recoverable, a botched fix is not.
const TIER_MODEL = { trivial: 'haiku', moderate: 'sonnet', intricate: 'opus' }
const tierFor = (key) => {
  const row = (triage?.groups ?? []).find((g) => g?.key === key)
  return TIER_MODEL[row?.tier] ?? 'opus'
}

const fixes = groups.length
  ? await pipeline(groups, (g) =>
      tryAgent(
        `Fix the review findings below. They were raised by a red-team fleet that attacked ` +
          `${featureBranch} and then tried to refute its own claims; everything here survived refutation. ` +
          `You did not write this code and you did not raise these findings.\n\n` +
          `Create a worktree at ${repoPath}/../wt-${g.slug} on new branch ${g.slug} based on ` +
          `${featureBranch}, and work only there. Do NOT touch ${featureBranch} itself and do NOT merge ` +
          `anything — a serial integrator lands your branch afterwards.\n\n` +
          `Findings, all in ${g.file}:\n${renderFindings(g.findings)}\n\n` +
          `For each: reproduce the failure from the code before changing anything. Make the smallest change ` +
          `that removes the defect and add or extend a test that would have caught it, unless the repo has no ` +
          `test for that area at all. A PLAUSIBLE finding turns on context the refuter could not settle — if ` +
          `reading the code shows it is not in fact a defect, skip it with a reason rather than changing ` +
          `working code. If a fix needs an irreversible or high-blast-radius action, or would require ` +
          `redesigning something the epic did not ask you to redesign, skip it and set hitl.triggered=true.\n\n` +
          `When done, squash to ONE commit on ${g.slug}, run whatever test/lint command the repo documents, ` +
          `and return the SHA, the ids you fixed, and the ids you skipped with reasons.`,
        { phase: 'Fix', schema: FIX_SCHEMA, label: `fix:${g.slug}`, model: tierFor(g.key) }
      )
    )
  : []

const fixed = []
const skipped = []
const hitl = []
const failedFixes = []
const ready = []

// Unfiltered index again: pipeline() returns one slot per group, nulling the dropped ones.
fixes.forEach((r, i) => {
  const g = groups[i]
  if (!r) {
    failedFixes.push({ key: g.key, file: g.file, ids: g.findings.map((f) => f.id), reason: 'fix agent died after retries' })
    return
  }
  ready.push({ ...g, ...r, model: tierFor(g.key) })
  ;(r.skipped ?? []).forEach((s) => skipped.push({ ...s, file: g.file }))
  if (r.hitl?.triggered) hitl.push({ key: g.key, file: g.file, reason: r.hitl.reason })
})

if (ready.length) {
  phase('Integrate fixes')
  const integration = await tryAgent(
    `You are the sole integrator for the fix branches below. Work in the primary checkout ${repoPath}, ` +
      `where ${featureBranch} is checked out. Handle them ONE AT A TIME, in the order listed.\n\n` +
      ready.map((r) => `- ${r.key} · branch ${r.branch} · commit ${r.commitSha} · file ${r.file} · fixed ${r.fixed.join(', ') || 'nothing'}`).join('\n') +
      `\n\nFor each, in order:\n` +
      `1. Re-verify the target with git -C ${repoPath} rev-parse --verify ${featureBranch}. The ONLY valid ` +
      `target is that exact branch — never ${PROTECTED_BRANCHES.join('/')}, never an invented fallback. If it is missing ` +
      `or wrong, STOP and report hitl.triggered=true for every remaining branch.\n` +
      `2. Skip any branch whose fixed list is empty and whose commit is unchanged from ${featureBranch} — ` +
      `report landed=false with that as the reason, not as a HITL.\n` +
      `3. Rebase the fix branch onto ${featureBranch}, squashed to one commit. On ANY conflict: ` +
      `git rebase --abort, leave the branch and worktree in place, set landed=false and ` +
      `hitl.triggered=true with the conflicting paths, move on. Never resolve a conflict yourself.\n` +
      `4. On a clean rebase: git -C ${repoPath} merge --ff-only <branch>, then remove the worktree ` +
      `(git worktree remove <path> --force).\n\n` +
      `Never push and never open a PR. Return one row per branch listed above.`,
    { phase: 'Integrate fixes', schema: INTEGRATE_SCHEMA, label: 'integrate:fixes', model: 'sonnet' }
  )

  const rows = integration?.results ?? []
  ready.forEach((r) => {
    const row = rows.find((x) => x?.key === r.key)
    if (!row) {
      failedFixes.push({
        key: r.key,
        file: r.file,
        ids: r.fixed,
        reason: `integrator returned no row — the fix commit ${r.commitSha} is safe on branch ${r.branch}; land it by hand`,
      })
      return
    }
    if (row.landed) {
      fixed.push({ key: r.key, file: r.file, ids: r.fixed, commitSha: row.commitSha, model: r.model })
    } else if (!row.hitl?.triggered) {
      // landed=false with no HITL is the benign case — the fixer changed nothing worth landing.
      failedFixes.push({ key: r.key, file: r.file, ids: r.fixed, reason: row.reason ?? 'not landed, no reason given' })
    }
    if (row.hitl?.triggered) hitl.push({ key: r.key, file: r.file, reason: row.hitl.reason })
  })
}

phase('Discrepancy check')
const discrepancies = await tryAgent(
  `Compare epic #${epic}'s spec (title, body, acceptance criteria) in ${repo} against what actually ` +
    `landed on ${featureBranch}. The epic's contribution is exactly git -C ${repoPath} diff ${baseSha}...HEAD ` +
    `and git -C ${repoPath} log ${baseSha}..HEAD — use that pinned SHA, not a branch name, so nothing landed ` +
    `on ${baseBranch ?? 'the base branch'} by anyone else during the run can leak into or out of your view. ` +
    `Read the closed sub-issues and their summary comments too. Report anything specified but not delivered, delivered but not ` +
    `specified, or diverging from the spec's intent. The branch has already been red-teamed for defects — ` +
    `you are checking coverage against the spec, not hunting bugs.`,
  {
    schema: { type: 'object', properties: { discrepancies: { type: 'string' } }, required: ['discrepancies'] },
    label: 'discrepancy-check',
  }
)

// Last call in the run: a bare `discrepancies.discrepancies` would throw away the entire report over
// one dead agent, after every fix had already merged. Degrade instead.
return {
  fixed,
  skipped,
  unjudged: unjudged.map((f) => ({ id: f.id, lens: f.lens, claim: f.claim, location: f.location })),
  hitl,
  failed: failedFixes,
  discrepancies: discrepancies?.discrepancies ?? 'UNKNOWN — the discrepancy-check agent died; re-run it manually.',
}
```

## Resuming after a partial failure

A fleet-wide API failure kills every in-flight agent at once, so the usual shape of a bad run is "all
implementations committed, integration dead" (Workflow A) or "all fixes committed, integration dead"
(Workflow B). Those commits are durable and survive on their branches — recover, don't restart:

1. **Read `journal.jsonl` in the run's transcript dir first.** It records each agent's actual return
   value, so it tells you which stages really completed. Don't infer it from the error, and don't
   assume a cached result is non-empty.
2. **Check every worktree's state** — `git -C <wt> status --porcelain` and `git -C <wt> log --oneline -1`.
   A fixer killed mid-edit leaves uncommitted changes; that partial work must be redone against the
   base, never committed on trust.
3. **Check the feature branch too.** The integrator is serial, so a mid-round death leaves a prefix of
   branches merged and the rest not. `git -C <repoPath> log --oneline` against the round's branch list
   tells you exactly where it stopped.
4. **Edit the persisted script, then resume**: `Workflow({ scriptPath, resumeFromRunId })`. The longest
   unchanged prefix of `agent()` calls replays from cache, so successful implementations cost nothing.
5. **Keep surviving prompts byte-identical.** The cache keys on `(prompt, opts)` in call order — reword
   a prompt and that call plus everything after it re-runs live. Wrapping `agent()` is safe; changing
   the string it receives is not.
6. **If the failure was rate/overload-driven, pass `retryAfterMs`** from whatever the provider asked
   for, and consider fewer sub-tasks per round.

Acts are separately resumable: if act 2's review completed and Workflow B died, re-run Workflow B
alone with the same `findings` — act 1's work is already on the branch.

## Notes on the design choices baked into these templates

- **Review is branch-wide and happens once, after everything has landed.** The earlier design put an
  Opus reviewer on each sub-task's diff immediately after its implementer. That review could not see
  the other sub-tasks — so the one class of defect a parallel fleet reliably produces, the interaction
  between two independently-correct changes, had no reviewer at all. Moving the review to the
  assembled branch also collapses N reviewer agents into one five-lens attack + refute pass, which is
  cheaper *and* harder on the code.
- **The fixed point is a pinned SHA, not a branch name.** `trunk` was hardcoded in an earlier draft,
  which is wrong in any repo that names its integration branch something else — but resolving the name
  correctly is only half the fix. A branch name is a moving reference: if anyone lands on the base
  while act 1 is building, `<base>...HEAD` re-scopes underneath the run, and the review either audits
  code the epic never wrote or skips code it did. So step 0 captures `git rev-parse <featureBranch>`
  *before the first sub-task lands* and every measurement — act 2's diff, act 3's discrepancy check —
  uses that SHA. `baseBranch` survives only as a name: for creating the branch, and for the protected
  set the integrator must refuse to merge into.
- **The refute stage is why fixes can be automated.** A per-sub-task reviewer that also fixed what it
  found was its own judge — nothing tested whether the finding was real before the code changed.
  `/adversarial-code-review` runs a fresh refuter per lens that never sees the attacker's reasoning, so
  by the time a finding reaches a fixer it has already survived a hostile second reading. `REFUTED`
  never reaches a fixer; `UNJUDGED` never does either, because a dropped verdict is not a survived one.
- **Fixers are tiered by fix complexity, not defect severity.** These are independent axes: a blocker
  can be a one-character fix and a minor smell can require reshaping an interface. Tiering on severity
  would systematically over-spend on the first and under-spend on the second. When the triage agent
  dies, the tier defaults to `opus` — over-spending on a fix is recoverable; a botched fix on a branch
  nobody will review again is not.
- **Integration is serial and never resolves conflicts.** Only one agent writes to the feature branch,
  one branch at a time, in the primary checkout. Concurrent `--ff-only` merges against a moving ref
  produce lost updates and conflicts that are artifacts of the schedule rather than the code. And an
  integrator that resolves conflicts is making semantic decisions about code it has not read, in the
  one place where a wrong guess is invisible — so a conflict aborts to HITL, always.
- **Grouping fixes by file is the concurrency unit.** Two fixers editing the same file in different
  worktrees produce two commits that conflict at integration, and the second one aborts to HITL for no
  reason but scheduling. Grouping by file makes those conflicts impossible by construction; only
  genuinely cross-file fixes can still collide.
- **Worktree lifecycle is scoped to one unit of work** (one sub-task, or one fix group), removed once
  its branch lands. This is why the claim/fix stages create worktrees explicitly with `git worktree
  add` rather than via `agent()`'s `opts.isolation: 'worktree'` — that option gives each *call* its own
  fresh worktree, which would hand a later stage a different checkout than the earlier one used.
- **`pipeline()` over `parallel()`** for claim/implement and for fixes: those units don't need to be
  synchronized with each other, so the slowest doesn't block the rest. The barrier before integration
  is the exception that proves the rule — a serial integrator genuinely needs the full set before it
  can order it (see the Workflow tool's own guidance on when a barrier is/isn't justified).
- **The outer `while` loop re-runs discovery** because landing one sub-task can unblock another (a
  dependency edge closes). The `MAX_ROUNDS` cap is a backstop, not a design target — it's logged when
  hit rather than silently truncating, since the caller needs to know coverage may be partial.
- **Blocking is a filter at Discover time, not a scheduler.** The Discover prompt must resolve
  children-of-epic and blocked-by exactly the way `docs/agents/issue-tracker.md` documents (or the
  plain-`gh` fallback) — don't leave it to the sub-agent to guess the convention. The same Discover
  call also excludes already-*assigned* issues, which is what keeps a sub-task claimed in an earlier,
  still-in-flight round from being re-claimed by a later round's pipeline — assignment doubles as the
  concurrency guard, dependency status doubles as the ordering guard.
- **Issues close at land time, not after review.** A finding raised in act 2 becomes a fix commit on
  the feature branch, never a reopened sub-task: the sub-task's contract was "this change, on this
  branch", and it met it. Keeping issues open through a branch-wide review window would leave every
  one of them assigned-and-open for the length of the slowest lens.
- **Every external input fails loud, not silent.** `args` is checked before a single agent spawns; the
  epic's existence is checked before Discover returns anything; the feature branch's existence is
  checked before Claim creates a worktree, and re-checked by the integrator before every merge. This is
  deliberate, not defensive boilerplate: a first run of this template hit exactly the failure mode
  these checks close — `args` arrived **JSON-stringified rather than as an object** (confirmed from the
  persisted run record: `args` was a `str` holding `'{"epic": 109, ...}'`). The keys were all correct;
  the container wasn't. On a string primitive every property read returns `undefined` rather than
  throwing, so every prompt silently interpolated the literal text `"undefined"`, and six review agents
  responded to an unresolvable merge target by defaulting to `trunk` — exactly the branch the skill's
  rules forbid touching — while a seventh, faced with the same broken input, correctly refused trunk
  but landed on an orphaned branch instead. A missing or malformed `args` field, or a merge target that
  isn't verifiably the real `featureBranch`, must stop the fleet (or that one unit of work) outright —
  never "good enough, proceed."
- **A dead agent must degrade one unit of work, never the fleet.** `agent()` returns `null` rather than
  throwing when a subagent dies on a terminal API error, and a provider-side **529 overload kills every
  in-flight agent simultaneously** — observed on this template's first real run, where four
  implementations were already committed and all four reviewers died at once. Inside `pipeline()` a
  throw is contained (the item drops to `null` and skips its remaining stages), which is why each stage
  null-checks its input and throws a descriptive error. What is *not* contained is anything after the
  pipeline: on that run a null result reached a property read in the result handler and the resulting
  `TypeError` aborted a workflow whose work was sitting safely on disk. So: null-check every stage, and
  let nothing outside `pipeline()` — result handler, integrator rows, or final discrepancy read —
  dereference an unchecked value.
- **A missing integrator row is reported, not assumed.** The integrator returns rows keyed by issue
  number / group key rather than positionally, and the result handler treats an absent row as a failure
  that names the surviving commit and branch. A single agent handling N merges is exactly where a
  partial answer is likely, and a dropped row must not read as "landed".
- **Back off; don't hammer.** `tryAgent` re-spawns a dead call up to three times with escalating waits
  from a 60s floor. A 529 means the provider is overloaded, so an immediate retry adds to the very load
  that caused the failure — and since the whole fleet dies together, every lane would retry in lockstep.
  The script can't read a `Retry-After` header (it never sees the HTTP response), so when the
  orchestrator observed one it passes `retryAfterMs` and that becomes the floor. Retries reuse a
  byte-identical prompt so `resumeFromRunId` still matches the cache.
- **Read `args` through the normalizer, never directly.** Each script destructures its fields once at
  the top and interpolates those bindings; `args` itself is referenced only in the error message.
  `args` is the sole `Workflow` input with no declared type in its schema, so nothing upstream
  guarantees it arrives as an object — a stringified value is a live possibility on any run, not a
  one-off. If you add a new field, destructure it up there too rather than reaching for `args.newField`
  mid-script, or you reintroduce the silent-`undefined` hole for that one field.
