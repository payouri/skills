# Reference — adversarial-code-review

Everything [SKILL.md](SKILL.md) points at: how each lens's rulebook resolves, and the workflow
script.

## Lens rulebooks

Each lens attacks from a rulebook. Three of them ship in [rulebooks/](rulebooks/), so this skill runs
standalone in any repo; Security delegates to a bundled skill, and Spec is inherently repo-specific.
Resolve each chosen lens into the `rulebook` instruction string the script passes to its attacker.

| Lens            | Rulebook                                                    | How the attacker gets it                 |
| --------------- | ----------------------------------------------------------- | ---------------------------------------- |
| Bugs            | [rulebooks/bugs.md](rulebooks/bugs.md)                      | resolved path, attacker reads the file    |
| Maintainability | [rulebooks/maintainability.md](rulebooks/maintainability.md) | resolved path, attacker reads the file    |
| Standards       | [rulebooks/standards.md](rulebooks/standards.md), plus this repo's documented rules | resolved path, plus discovered repo docs |
| Security        | built-in `security-review` skill (bundled, not a file)      | attacker invokes the skill                |
| Spec            | the originating issue / PRD                                 | fetched contents, or a resolved path      |

Resolve this skill's own directory once — it may be linked from a project or a global skills dir:

```bash
for d in .claude/skills .cursor/skills ~/.claude/skills ~/.agents/skills; do
  ls -d "$d"/adversarial-code-review/rulebooks 2>/dev/null
done | head -1
```

**Bugs**, **Maintainability** — instruction: `Read <rulebooks>/<lens>.md and attack this diff by
every rule it lays down — it is your rulebook.`

**Security** — always available; the skill ships with the CLI. Instruction: `Invoke the
security-review skill and apply it to this diff.`

**Standards** — also collect what this repo documents: `CLAUDE.md`, `AGENTS.md`,
`.claude/rules/*.md`, `CONTRIBUTING.md`, `docs/adr/`, app-scoped `docs/adr/`, `DESIGN.md`,
`architecture.md`. Instruction: `Read <paths>, then <rulebooks>/standards.md. The repo's documented
rules bind hardest; the smell baseline applies on top, and a documented repo rule always overrides
it. Skip anything tooling already enforces.`

**Spec** — resolve in order: issue references in the commit messages (`#123`, `Closes #45`, a
ClickUp id) fetched through the tracker workflow this repo documents; a path the user passed; a
PRD/spec under `docs/`, `specs/`, or `.scratch/` matching the branch or feature; then ask the user.
Instruction: `The spec is <path, or the contents pasted>. Attack the gap between it and the diff.`
No spec → Spec is unavailable; say so rather than inventing the requirements.

**Vendored, not linked.** `maintainability.md` and `standards.md` are copies — of
`thermonuclear-review/SKILL.md` and `code-review`'s Fowler smell baseline respectively, provenance in
each file's header. Copying is what makes this skill standalone, and staleness is the price: when
either upstream changes, refresh by re-copying its body.

## Workflow script

`args`:

```json
{
  "diff": {
    "command": "git diff <fixed-point>...HEAD",
    "log": "<pasted output of git log <fixed-point>..HEAD --oneline>"
  },
  "lenses": [
    { "name": "Bugs", "question": "does it break?", "rulebook": "<instruction string>" }
  ]
}
```

```js
export const meta = {
  name: 'adversarial-code-review',
  description: 'Attack a diff one lens at a time, then refute every finding before reporting it',
  phases: [
    { title: 'Attack', detail: 'one adversarial finder per lens' },
    { title: 'Refute', detail: 'one fresh refuter per lens, over that lens claims only' },
  ],
}

const FINDINGS = {
  type: 'object',
  additionalProperties: false,
  required: ['findings'],
  properties: {
    findings: {
      type: 'array',
      maxItems: 8,
      items: {
        type: 'object',
        additionalProperties: false,
        required: ['id', 'claim', 'location', 'hunk', 'failureScenario', 'severity', 'rationale', 'fix'],
        properties: {
          id: { type: 'string', description: 'short slug, unique within this lens' },
          claim: { type: 'string', description: 'one sentence stating the defect' },
          location: { type: 'string', description: 'path:line' },
          hunk: { type: 'string', description: 'the exact changed lines the claim is about' },
          failureScenario: { type: 'string', description: 'concrete input or state -> wrong outcome' },
          severity: { type: 'string', enum: ['blocker', 'major', 'minor'] },
          rationale: { type: 'string', description: 'the reasoning that convinced you' },
          fix: { type: 'string', description: 'smallest change that removes the defect' },
        },
      },
    },
  },
}

const VERDICTS = {
  type: 'object',
  additionalProperties: false,
  required: ['verdicts'],
  properties: {
    verdicts: {
      type: 'array',
      items: {
        type: 'object',
        additionalProperties: false,
        required: ['id', 'verdict', 'reason'],
        properties: {
          id: { type: 'string' },
          verdict: { type: 'string', enum: ['CONFIRMED', 'PLAUSIBLE', 'REFUTED'] },
          reason: { type: 'string', description: 'one line; for REFUTED, what killed it' },
        },
      },
    },
  },
}

// `args` arrives as a JSON string on some hosts and as an object on others; parse defensively or
// `pipeline(input.lenses)` throws before a single agent spawns.
const input = typeof args === 'string' ? JSON.parse(args) : args

const attack = (lens) => `You are red-teaming a code change on ONE lens: ${lens.name} — ${lens.question}

${lens.rulebook}

The change: \`${input.diff.command}\`
Commits:
${input.diff.log}

Read the diff, then read enough of the surrounding code to know how the changed lines behave in
context — the diff alone hides callers, defaults, and which paths reach them.

Now attack. Assume the author shipped under time pressure and that something here is wrong. For each
finding, build the concrete case that breaks it: the specific input or state, and the wrong outcome
it produces. A finding you cannot ground in a concrete case is not a finding — drop it.

Report at most 8 findings, highest conviction first, each anchored to a path:line and the quoted
hunk. Put your argument in \`rationale\`: it is withheld from the reviewer who will try to refute
you, so \`claim\` should state the defect and \`rationale\` should carry the case for it.`

const refute = (lens, claims) => `Fresh eyes on a code change. Someone claims the defects below on the ${lens.name} lens. You have their claims and nothing else — their reasoning is deliberately withheld, so reach your own.

The change: \`${input.diff.command}\`

Claims:
${claims.map((c) => `- id: ${c.id}\n  claim: ${c.claim}\n  at: ${c.location}\n  hunk: ${c.hunk}\n  they say it fails when: ${c.failureScenario}`).join('\n')}

Read the code yourself and try to make each claim fail:

- REFUTED — you can show the claim is wrong: the code already handles it, the path is unreachable,
  the quoted hunk does not do what the claim says, or the rule or requirement it cites says no such
  thing. Your reason is the one line that kills it.
- CONFIRMED — you rebuilt the failure from the code yourself and could not break the claim.
- PLAUSIBLE — the concern is real, but the code cannot settle it; it turns on runtime behaviour,
  data, or context you do not have.

CONFIRMED means you reproduced the reasoning, not that the claim sounds right — a claim you can
neither reproduce nor kill is PLAUSIBLE. Return exactly one verdict per id above, inventing no ids.`

const results = await pipeline(
  input.lenses,
  (lens) => agent(attack(lens), { label: `attack:${lens.name}`, phase: 'Attack', schema: FINDINGS }),
  (found, lens) => {
    const findings = found?.findings ?? []
    // The wall: rationale, fix, and severity never reach the refuter.
    const claims = findings.map(({ rationale, fix, severity, ...claim }) => claim)
    if (!claims.length) return { lens: lens.name, findings: [] }
    return agent(refute(lens, claims), { label: `refute:${lens.name}`, phase: 'Refute', schema: VERDICTS }).then((v) => ({
      lens: lens.name,
      findings: findings.map((f) => {
        const judged = (v?.verdicts ?? []).find((x) => x.id === f.id)
        return { ...f, verdict: judged?.verdict ?? 'UNJUDGED', verdictReason: judged?.reason ?? '' }
      }),
    }))
  },
)

return results.filter(Boolean)
```
