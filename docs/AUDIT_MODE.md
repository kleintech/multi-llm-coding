# Audit mode — pointing a panel at a whole codebase

`crossexam review` judges a diff. `crossexam audit` judges a **standing codebase**: architecture,
documentation conformance, invariants, accumulated drift. Different unit, different context
strategy, different verification story.

This is the mode for "what's wrong with what I already have," as opposed to "don't let this PR make
it worse."

---

## 1. Why "review my architecture" is not a task

A representative target: **128K LOC across 657 source files** — roughly 1.5–2M tokens. Three
consequences:

1. **The repo does not fit**, in any model, at any price. And "fits" is the wrong bar anyway: recall
   degrades well before the context limit, so a 1M-token prompt buys attention dilution rather than
   understanding.
2. **There is no natural unit.** A diff bounds itself. A codebase does not. Without a unit there is
   nothing to cluster, nothing to verify, and nothing to say "done" about.
3. **The prompt is unfalsifiable.** "Review the architecture" gets architecture-shaped prose back,
   confident and unactionable — the same failure as "is this good UX?"

So audit mode does not review a codebase. It answers **bounded claims about a codebase**, one at a
time, in parallel.

## 2. The unit: a claim

Four claim generators, in descending order of value-per-token:

### 2a. Doc↔code drift *(the highest-value one, and it is nearly free)*

Every ADR and design doc is a **falsifiable assertion about the code**. A repo with 24 ADRs and 25
design documents is carrying 49 testable claims, and some fraction of them silently stopped being
true. Nobody notices, because nothing checks.

```
claim:  docs/decisions/0003-tenant-boundary.md
ask:    Does the code still satisfy every normative statement in this ADR?
        For each: SATISFIED / VIOLATED / SUPERSEDED / UNVERIFIABLE, with file:line.
```

This is the best unit available because it is bounded (one document), falsifiable (the document
says what should be true), and it targets exactly where value decays invisibly. It also has a
documented failure mode behind it: the class-2 false positive in
[`OBSERVED_FAILURES.md`](OBSERVED_FAILURES.md) was a reviewer misreading which *phase* of an ADR's
migration a diff implemented. Auditing an ADR directly, with the whole document in context, is the
setting where that reasoning actually has what it needs.

`UNVERIFIABLE` is a first-class outcome, not a failure. A doc statement no reviewer can check is
either too vague to be a standard or describes something unobservable — both worth knowing.

### 2b. Invariant sweep

A single rule, checked everywhere: *"every API route scopes queries by `tenantId`," "every list
route clamps pagination," "no route reads a session without `resolvePrincipalFromSession`."*

Mechanically checkable, high-signal, and the output can ship as a **lint rule** (§5). Seed these
from `ENGINEERING-PRINCIPLES.md` and from ADR normative statements the drift pass marked
`SATISFIED` — those are exactly the invariants worth locking down before they rot.

### 2c. Undocumented-invariant discovery

The mirror image, and the one your bug history argues for.
[`ilm-realtor-525`](../bench/corpus/ilm-realtor-525.md)'s defect was violating an invariant —
*`MediaAsset.url` must live in the ambient Blob store* — that **is written down nowhere.** It exists
implicitly in an untokened `del()` call.

```
ask:  In this module, what does the code assume that no doc, comment, or type states?
      For each: the assumption, where it is relied on, and what breaks if violated.
```

Output is a candidate ADR list, not findings. Recording an invariant converts it from a trap into
something 2a can audit forever after. This is the only mechanism proposed anywhere in this project
that addresses the ceiling `ilm-realtor-525` identified.

### 2d. Open question

Operator-supplied: *"where is tenant isolation weakest?"* Least bounded, most useful when you
already suspect something. Keep it a minority of any run.

## 3. Context: the repo map, then exploration

Two-phase, because pre-computing the right context is impossible when the relevant code depends on
what the reviewer finds.

**Phase 1 — the map** (deterministic, cheap, cached, shared by all reviewers):

- module/dependency graph, fan-in and fan-out per module
- LOC and 12-month churn per area (`git log`), which localizes risk
- test presence and coverage per area
- route table, schema summary, entry points
- **doc index**: every doc and ADR with its title, status, and which modules it names

The map is orientation, not evidence. It is the audit-mode analogue of the packet and it is what
makes bounded exploration efficient rather than random.

**Phase 2 — read-only agent leaves.** Each reviewer gets the map, its claim, and a **read-only
toolset** (`grep`, `read`, `list`) with a hard call budget (default 40). It explores, gathers
evidence, returns findings against the schema.

This is a real departure from [ADR-0001](adr/0001-deterministic-orchestration.md) and is recorded
as [ADR-0007](adr/0007-read-only-agent-leaves.md). The orchestrator stays deterministic — fan out
claims × models, collect, cluster, adjudicate — so run structure remains reproducible and
worst-case cost stays computable as `claims × models × budget × max_tokens`. Only the *content* of
a leaf is adaptive. ADR-0001 explicitly anticipated this shape: *"it belongs inside a leaf … not in
the orchestrator."*

## 4. Verification

Better than [UX judgment](UX_REVIEW.md), worse than diff review. The same measurable/judgment split
applies, and for architecture a **larger** share falls on the measurable side:

| Measurable | Judgment |
| --- | --- |
| Circular dependencies; dead exports; modules with no tests; a doc claim contradicted at a specific line; "all N routes do X" where the count is checkable; schema↔code divergence; an invariant with enumerable violations | "This abstraction is wrong"; "these two modules should be one"; "this layering is confused" |
| Gated, reported as fact | Collected, triaged, never gated |

T0 transfers directly: a finding must cite a real file and line, and quoted code must actually
appear. A drift claim that cannot point at the violating line is dropped, exactly as a fabricated
citation is.

**Judgment-class architecture findings are the weakest output this project produces.** They are
unfalsifiable, models are fluent at generating them, and cross-family agreement mostly reflects
shared training data. Cap them hard, mark them as opinion, and expect to disagree with most.

## 5. The best output: a finding that ships as a check

Audit mode's analogue of *execution beats consensus*.

A drift or invariant finding with enumerable violations can be emitted as a **failing check** — an
ESLint rule, a `ripgrep` assertion in CI, a schema test, a custom AST rule:

```
ADR-0003 §"Tenant boundary": every Prisma query in src/app/api/** filters by tenantId.
  3 violations: routes/x.ts:44, routes/y.ts:91, routes/z.ts:12
  → proposed check: eslint-local-rules/require-tenant-filter.js (fails on exactly these 3)
```

That converts a one-time opinion into a permanent guarantee, and it is the only form of audit output
that keeps paying after you read it. Prioritize claims that can produce one.

## 6. What this is *not*

**Not a replacement for interactive exploration.** "Point another model at my codebase and talk to
it" is a different want, and the right tools already exist: Codex CLI, Gemini CLI, Cursor, or the
`llm` CLI wrapper. Use those to *steer*. crossexam audit is a **batch, decorrelated second opinion**
that returns a report — you then work through the report interactively, in whatever agent you like.

The composition that makes sense: Claude Code (or any agent) for building and steering; audit mode
on a schedule for the sweep no interactive session will ever do systematically, from models whose
blind spots differ from your daily driver's.

**Not run against untrusted code.** `SECURITY.md`'s "reviewers have no tools" mitigation exists
because PR content is attacker-controlled. Audit mode reverses that: reviewers *do* get tools, and
the justification is that the target is **your own repository**. The tools remain read-only with no
network and no writes, but pointing audit mode at an untrusted dependency's source reintroduces a
threat model it was not designed for.

## 7. Cost and cadence

Per-claim cost is bounded by the tool budget, so a run is `claims × models × ~40 calls`. A 49-claim
doc-drift sweep across two families is a few hundred bounded agent calls — meaningful but not
alarming, and it is a **quarterly** activity, not a per-PR one.

Sensible cadence: doc drift quarterly or on any ADR edit; invariant sweeps in CI once they have been
converted to checks (§5); undocumented-invariant discovery once per subsystem, then on major
refactors.
