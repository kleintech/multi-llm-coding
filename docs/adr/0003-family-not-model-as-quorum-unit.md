# ADR-0003: The model *family* is the quorum unit, not the model

**Status:** Accepted (design phase)

## Context

"Three reviewers agree" is only evidence if the three reviewers are independent. Three
`gpt-5.6-*` variants agreeing is close to one reviewer answering three times — same pretraining,
same post-training, same failure modes. Yet the natural implementation counts reviewers.

## Decision

Every model is assigned to a declared **family**. Independent-family count — not reviewer count —
is the input to clustering signal strength, refuter eligibility, and the publication table. Two
reviewers in one family are one voice everywhere.

A `families.yaml` file ships with the tool and is maintained as a data file, with glob matching.

Corollaries:

- **Refuter exclusion.** A family cannot refute a finding it raised (no self-corroboration and no
  self-defense).
- **Recusal.** The diff author's family is excluded from refutation entirely.
- **Config validation** refuses a panel with fewer than two families.

## Rationale

The entire premise of the tool is decorrelation. Counting reviewers instead of lineages would let
a user assemble a panel of five OpenAI models, see `independent_families: 5` on every finding, and
believe they had bought independence they do not have. That is not a small bug — it is the tool
lying about the thing it exists to provide.

Recusal follows directly: a model asked whether its own output is correct is not an adversary.

## Consequences

- Family assignment is a **judgment call** with no ground truth, especially for open-weights models
  fine-tuned from another lab's base. Rule: **when lineage is unclear, merge rather than split.**
  Wrongly merging costs a little diversity. Wrongly splitting silently inflates every
  independent-family count in the tool.
- `families.yaml` needs maintenance as models ship. An unknown model gets its own family *and a
  warning*, never a silent fold-in.
- Users who want a many-variant single-vendor panel get less from crossexam than they expect. That
  is correct behavior, and `panel explain` should say so out loud rather than let them find out
  from a bad review.
