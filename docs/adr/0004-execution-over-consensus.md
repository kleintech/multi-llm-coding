# ADR-0004: Execution outranks consensus

**Status:** Accepted (design phase)

## Context

Given a finding, we can ask more models whether they agree, or we can try to *run* something that
settles it. Multi-model review tools almost universally do the former: they are vote aggregators.

## Decision

Findings are pushed through a ladder that prefers facts to opinions:

- **T0** static reference check (free) — drops findings citing code that doesn't exist
- **T1** sandbox execution of a reviewer-supplied repro — produces `CONFIRMED_EXECUTABLE` /
  `REFUTED_EXECUTABLE`
- **T2** model refutation — only when T1 is unavailable

Executable verdicts outrank consensus verdicts in both directions. A passing counter-repro from a
refuter kills a finding that three families agreed on.

Reviewers are *required* to supply a repro for `behavioral` claims, and `claim_type` is a
first-class schema field precisely so the ladder can route on it.

## Rationale

- Model agreement is correlated error as often as it is signal. Execution is not.
- T0 is free and expected to remove the largest single false-positive class — fabricated citations.
  Running it before the expensive T2 stage is both a precision win and a cost win.
- Requiring a repro changes reviewer behavior upstream: a model asked to produce a failing test
  cannot fall back on vague architectural discomfort. The demand for evidence is itself a filter.
- It gives PR readers something no vote can: "here is the test that fails."

## Consequences

- Requires a sandbox, which requires a container runtime. Where none exists, **T1 is skipped and
  the run says so** — never silently downgraded to consensus while keeping confident labels.
- Concurrency and performance bugs are hard to reproduce deterministically and will mostly resolve
  at T2. Accepted; they are also where cross-model consensus is most useful.
- **The main risk:** a repro that fails for a reason unrelated to the claimed defect yields the
  system's highest-trust verdict on a bogus finding. Mitigated by requiring the repro to reference
  the cited symbol and by recording full repro output for audit, and tracked as a dedicated bench
  metric. This is the design's most credible remaining false-positive path and it should be
  measured before `block_on` is recommended to anyone.
- Repro repair loops are capped at one round-trip; repair loops are where cost runs away.
