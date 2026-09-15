---
name: research-dossier
description: Compile any topic into an evidence-graded dossier — guide, rulebook, sources — filed under one directory of the research repo and summarised in its README.
disable-model-invocation: true
argument-hint: "<topic> [target repo path]"
---

# Research dossier

A **dossier** is a compiled file of evidence on one subject: everything found, graded by how much
weight it can bear, with the disagreements and the gaps on the record rather than smoothed away.

Every dossier has the same shape, whatever the topic: four **axes** of research, three files, one
grading scale. That sameness is the point — a reader who knows one dossier knows them all, and a
claim's weight is legible without re-reading the sources.

## 1. Frame the axes

Instantiate each axis as the specific question it answers for this topic:

| Axis | What it asks |
|---|---|
| **Primary** | What does the authority actually say — the spec, standard, vendor docs, canonical source? What does it mandate versus leave open? |
| **Implementations** | How does it behave in practice, across every implementer? Where do they diverge from the authority and from each other? |
| **Corpus** | What do real artifacts look like in the wild? Frequency, distribution, exemplars, cautionary cases. |
| **Evidence** | What has been measured? Research literature, benchmarks, failure modes, security. |

Some topics leave an axis genuinely empty — a topic with no artifacts has no corpus. Drop it and
carry the one-line reason forward; it belongs in the dossier's own account of its limits.

**Done when:** four axes are named with their topic-specific question, and any dropped axis has its
reason recorded.

## 2. Fan out

One agent per axis, all dispatched in a single message so they run concurrently. Read
[REFERENCE.md](REFERENCE.md#the-fan-out-brief) before writing the prompts — the brief is the contract
that makes four independent reports mergeable, and an agent briefed loosely returns prose that cannot
be graded.

**Done when:** every axis has returned a report whose claims carry URLs and dates. A report that
gestures at sources without naming them goes back to its agent with the gap named.

## 3. Reconcile

The reports are four witnesses, not four chapters. Read them against each other.

- Where two disagree, resolve it against the higher **tier** and record how — one agent's
  unverified second-hand claim is often another's confirmed **primary** source.
- Where the authority contradicts its own implementers, that is not noise to average out. It is
  usually the most useful finding in the dossier.
- Where no report could source a claim that circulates widely, it goes on the record as folklore.

**Done when:** every disagreement between two reports appears in the conflicts list marked resolved
or open, and every unsourceable claim appears in the unverified list.

## 4. Write the dossier

Three files, in the topic's directory. Each has a fixed job:

**`guide.md`** — the synthesis. What is true, what follows from it, why it matters. Prose that argues
a position, quoting **verbatim** where the exact wording carries the weight. Leads with what the
authority actually is, then what the evidence supports, then what practice looks like, then failure
modes. Ends with the shape of the advice: what a reader should do.

**`rulebook.md`** — numbered, checkable rules distilled from the guide, grouped by the decision each
serves. One rule per line-item, each carrying its test (how a third party checks compliance), its
tier, and its source key. Ends with a short review checklist naming the rules that gate a pass.

**`sources.md`** — every source, keyed for citation, tiered, dated, with one line on what it settles.
Grouped by axis. Ends with two sections that are the dossier's integrity: **conflicts resolved**
during the research, and what remains **unverified**.

Grade every claim by the weight its source can bear:

| Tier | Source |
|---|---|
| **P1** | Primary specification or vendor documentation |
| **P2** | Peer-reviewed or preprint research |
| **P3** | Vendor engineering blog or industry research with disclosed method |
| **P4** | Practitioner report carrying measurement |
| **P5** | Opinion, anecdote, or unverified secondary claim |

A tier is not a compliment. A P1 that contradicts observed behaviour and a P5 that predicts it both
get said, with their tiers attached.

**Done when:** all three files exist; every rule carries a test, a tier and a source key; and every
source key cited in `guide.md` or `rulebook.md` resolves to an entry in `sources.md`.

## 5. File it

The dossier lives in one directory of the research repo, named for the topic in whatever style its
sibling directories already use. Then the root `README.md` earns its keep: a section for the dossier
summarising its headline findings — each one linking to the source that establishes it — above links
to the three files.

Confirm the push landed and report the URL.

**Done when:** the topic directory sits beside its siblings, the README summarises this dossier's
findings, and the remote has the commit.

## Refreshing a dossier

A dossier is dated, so it goes stale. See
[REFERENCE.md](REFERENCE.md#refreshing-an-existing-dossier) for how to re-run one against its
own record.
