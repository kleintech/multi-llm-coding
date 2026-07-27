# ADR-0005: Precision over recall, enforced by hard caps

**Status:** Accepted (design phase)

## Context

A panel of frontier models on a moderate diff will produce 20–60 raw findings. Publishing them
optimizes recall. It also guarantees the tool is muted within a week — the failure mode of every
review bot that has ever been switched off.

## Decision

- Publish only findings that clear the verification ladder and the publication decision table.
- Hard caps: **10 inline comments per run, 3 per file, 5 advisory entries.** Overflow moves to the
  summary with an explicit count, never silent truncation.
- `design`-type claims are never inline.
- Default review action is `COMMENT`, never `REQUEST_CHANGES`. Blocking is opt-in and available
  *only* for findings backed by a test the tool actually ran.
- Dropped and refuted findings are shown in a collapsed section, not hidden.
- Every published finding carries a provenance footer stating whether it was proved or voted on,
  and by how many families.

Target: **precision ≥ 0.85 at published**, treated as a release gate rather than an aspiration.

## Rationale

The costs are asymmetric and the asymmetry is severe. A missed bug costs one bug. A false positive
costs a maintainer's attention, and enough of them cost the tool's existence — after which every
subsequent bug is also missed. Recall is worth nothing from a disabled tool.

Caps additionally encode a real signal: a PR with thirty genuine problems is a PR that should be
split, and the useful output is "here are the ten worst," not thirty annotations nobody reads.

Showing the dropped findings, rather than hiding them, is what makes the filter auditable — and
an unauditable filter never gets tuned, because nobody can see what it is doing wrong.

## Consequences

- crossexam will miss real bugs it actually found. This is chosen, not accidental, and the bench
  measures exactly how often (recall @ seeded critical) so the trade stays visible.
- Cap tuning is a free offline sweep over saved run records — the caps can be re-tuned against the
  bench without spending a dollar.
- On very large diffs the caps bind hard. The right response is a "this PR is too large to review
  well" notice, which is honest and probably true.
- If dogfooding still feels noisy, **lower the caps before lowering the thresholds.** Fewer
  comments at the same quality bar beats more comments at a lower one.
