# Research dossier — reference

Disclosed reference for [`research-dossier`](SKILL.md): the brief that makes four reports mergeable,
how to refresh a stale dossier, and a worked example.

## The fan-out brief

Four agents research independently and never see each other's work, so the brief alone decides
whether their reports can be reconciled. Give each one the same eight clauses, varying only the
axis-specific investigation list.

1. **The axis, as a question.** Not a topic — the specific question this agent owns. Overlapping
   briefs return overlapping prose; disjoint briefs return a dossier.
2. **Live sources, fetched now.** Name the tools and require them: `WebSearch` and `WebFetch`, loaded
   via `ToolSearch` first. An agent answering from memory returns a plausible dossier of things that
   were true a year ago, and nothing in the report marks which parts those are.
3. **Primary over commentary.** Send the agent to the spec, the vendor docs, the paper, the source
   file — and to the repository's own source when documentation and behaviour might diverge. A
   configuration key read out of the code settles what a blog post can only assert.
4. **A verbatim quote per claim.** Short, exact, in quotation marks. Paraphrase is where accuracy
   goes to die: it survives the report, loses its hedges, and arrives in the dossier as a fact.
5. **A date on every source**, plus a note where a URL redirected or moved. Redirect chains are how
   you discover a vendor reorganised its docs and half the internet's advice now cites a 404.
6. **Explicit uncertainty.** Require the agent to separate what it verified from what it inferred,
   to label opinion as opinion, and to say plainly where it could not confirm something. An agent
   permitted to say "unverified" stops guessing.
7. **Conflicts named, not resolved.** The agent flags where its own sources disagree and leaves the
   resolution to reconciliation, which is the only place with all four reports in view.
8. **A structured Markdown report as the final message, and no files written.** Tables where they
   help, a `Sources` section listing every URL with title, publisher and date. The final message is
   the whole deliverable — reconciliation reads it, not a scratch file.

Tell each agent not to read the local repository unless its axis is about that repository. An agent
that starts reading local code drifts from research into code review.

## Reconciling four reports

The reports arrive with predictable seams:

- **A number one agent could not verify and another confirmed from source.** Take the primary, note
  it in conflicts resolved. This is the most common seam and the reason four axes beat one.
- **A claim that circulates in every secondary source with no primary behind it.** Folklore. Say so,
  and say what direction the evidence does support.
- **A hallucinated detail** — an entry on a list, an attribution, a repository path. If one report
  asserts what another's fetched source contradicts, the fetched source wins and the correction goes
  in the record.
- **Numbers pulled from an abstract rather than the paper.** Keep them, mark the provenance.
- **Two measurements pointing opposite ways.** Do not average them. State both, state the plausible
  reason they differ, and mark it unresolved.

## Refreshing an existing dossier

A refresh is a re-run against the dossier's own record, and it inherits the four axes already framed.

1. Read the existing `sources.md` first — the conflicts and unverified sections are the agenda. Every
   entry there is a question the last run left open.
2. Brief the agents with those open questions named explicitly, alongside their axis, and with the
   dossier's research date so they can hunt for what changed since.
3. Re-check the P1 sources by fetching them again. Vendor documentation moves, gets reorganised, and
   quietly reverses itself; a quote that no longer exists at its URL is the highest-value finding a
   refresh produces.
4. Update all three files, and add what changed and what was retired to the record. A dossier that
   silently drops a retracted claim loses the reader's ability to trust the ones that remain.
5. Re-date the dossier and refresh the README summary.

## Worked example: the `agentsMd` dossier

The first dossier, in `payouri/agentic-research`, researching what a state-of-the-art `AGENTS.md`
should be. Its axes instantiated as:

- **Primary** — what the `agents.md` standard mandates versus leaves open; placement, nesting,
  precedence, size, relationship to README. *Finding: there is no specification at all — no schema,
  no required headings, `/spec` returns 404. The entire normative surface is four FAQ answers.*
- **Implementations** — how each of 23 tools discovers and merges the file. *Finding: the standard's
  own "nearest file wins" rule holds in 2 tools; 12 concatenate, 2 load exactly one file, 1
  guarantees no order. Also the binding size cap, read out of a config key: 32 KiB combined.*
- **Corpus** — 37 real files from notable public repos, with section-frequency and length
  distributions, and quoted exemplars against cautionary cases. *Finding: median ~155 lines, and the
  highest-value section is the one nobody writes — "here is the thing you will think is a bug".*
- **Evidence** — measurements, instruction-following literature, failure modes, injection. *Finding:
  two 2026 studies split cleanly — no gain in task success, large gain in efficiency — and that split
  is what the guide is built around.*

What the run demonstrates: the reconciliation step earned its place four separate times (an
unverified cap confirmed from primary source, a hallucinated compatibility-list entry caught, a
widely-repeated instruction-count figure found to have no primary source, and two cost measurements
left explicitly unresolved), and the finding that ended up leading the guide came from an axis
disagreeing with the authority rather than from any single report.
