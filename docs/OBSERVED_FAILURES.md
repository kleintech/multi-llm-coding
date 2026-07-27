# Observed failures

Real false positives from the predecessor system (`llm-delegate` + `/adversarial-review` in
`kleintech/dotfiles`), classified. This document exists because the rest of the design was
originally built on an *assumption* about what false positives look like, and this is the only
page in the repository backed by observation rather than reasoning.

**Sample: 4 false positives, 3 runs, 1 pull request, 1 adversary model (`gemini`).** That is small
enough that the proportions below mean little. What it does establish is that **at least four
mechanistically distinct classes exist**, which is enough to falsify a single-class design.

## Setup that produced them

| | |
| --- | --- |
| Target | A **clean, already-merged** PR — "feat(identity): ADR 0022 expand" |
| Adversary | `gemini` (single model, single persona, free-text output) |
| Context | Bundled changed files + imported files; later runs added `schema.prisma`, then PR title/body |
| Verifier | Claude, in-context, by hand |

## The classes

### 1. Absence claim — falsified by a file outside the bundle

Two findings about null foreign keys, resolved by adding `schema.prisma` to the bundle. The model
claimed a constraint was not guaranteed; the guarantee lived in a schema file it had never seen.

*Fixed by:* more context. **Vindicates the T0.5 / symbol-slice line of attack** — and note the
falsifier was a *schema*, not a function definition, which a naive symbol slice would miss.

### 2. Intent misattribution — the model inferred the wrong *purpose*

> "Re-introduces the deprecated `splitDisplayName` fallback, contradicting ADR 0022."

Rejected. The PR implements **step 1 (Expand)** of a documented expand→switch→contract migration,
where retaining the shortcut is correct. Step 3 (Switch) is where it gets dropped. The model
conflated the phases and read an ADR-sanctioned temporary as a violation.

The decisive missing context was **the PR title** — the word "expand." Notably the ADR *was*
already in front of the model and it still mis-mapped the step, so this is partly irreducible: a
multi-hop "which phase does this diff implement?" inference is exactly where models slip.

*Fixed by:* feeding change intent first, plus prompt discipline. **This class is not addressed by
any amount of code context**, and the original design under-served it — see §Design impact.

### 3. Failure to trace context already provided

> "Should recompose name from first/last instead of using `input.client.name`."

Rejected. The call sites that disprove it were **changed files in the same PR** — already in the
bundle. The model simply did not trace `input.client.name` back through them.

*Fixed by:* nothing context-related, because nothing was missing. Only refutation by a second model,
or better prompting, touches this class. **T0.5 would do nothing here**, and a repo-wide search
would return matches that look like corroboration.

### 4. Nit inflation — a style preference dressed as a correctness defect

The same finding was, on inspection, "at most a repo-wide defensive style choice — out of scope."
The prompt already banned style nits. The model relabeled one as a correctness concern to get past
that ban.

*Fixed by:* prompt discipline, partially. **Self-assigned severity and claim type are not
trustworthy**, which has direct consequences for a schema that asks models to self-assign both.

## The test-selection trap

The most important observation from the predecessor runs is not about any finding.

**All three runs targeted a clean, already-merged PR.** On a clean PR a correct adversary emits
exactly one thing: nothing. So the only possible outputs were false positives and silence — a true
positive was *structurally impossible*. Across three rounds the adversary contributed zero real
findings, and every real judgment came from the human-side verifier.

It is easy to read that as "the adversary is worthless." It is not evidence for that at all. It is
an artifact of choosing a test case with no bugs in it, and it means the predecessor system's
*recall was never measured even once* — only its false-positive rate.

This is a bench-design rule, not a footnote: **any evaluation corpus must contain known defects, or
it can only measure precision.** It is why `EVALUATION.md` builds seeded bugs and historical
bug-fix pairs before anything else, and why the roadmap puts the bench at M1.

Corollary worth stating: over-tuning against a clean PR optimizes purely for silence. Every prompt
change that suppresses output looks like an improvement, and a model tuned to say nothing scores
perfectly.

## Design impact

| Observation | Change |
| --- | --- |
| Four distinct classes, only one is absence-shaped | ADR-0006's premise narrowed. T0.5 addresses class 1 only, and is sized accordingly. |
| Falsifier was a schema file | Slicing must cover schema/migration/config/IDL, not just code symbols. |
| Intent was decisive (class 2) | `intent` promoted to a **required, first-position** packet section, with an `intent` persona brief that reads it before the diff. Resolves against the security fencing — see below. |
| Model had the data and didn't trace it (class 3) | Named as its own class. Context mechanisms explicitly do not claim to fix it; cross-family refutation is the only lever. |
| Nit relabeled as correctness (class 4) | Self-assigned `claim_type`/`severity` treated as untrusted input, cross-checked rather than believed. |
| Clean-PR-only testing | Bench rule: corpora must contain known defects. Recorded in `EVALUATION.md`. |

### The intent tension

Class 2 says intent is decisive context. `SECURITY.md` says the PR description is
attacker-controlled and must be fenced as untrusted. Both are true, and they are compatible only
if stated precisely:

- Intent is placed **first** in the packet, because the model must read it before the diff to scope
  what the change is trying to do.
- It is fenced as **untrusted**, because it cannot be allowed to redirect the reviewer's task.
- The distinction that makes this work: intent is admissible as evidence about *scope and phase*
  — "which step of a migration is this?" — and inadmissible as *instructions* or as a claim that
  the code is correct.
- The residual risk is real and accepted: an attacker can mislead a reviewer about scope via the
  PR description. That is a quality attack, not an escalation, and it is exactly what the `intent`
  persona is briefed to catch by checking the description *against* the diff.

Anonymization (`ARCHITECTURE.md` §2.3) strips authorship signals only. It must not strip intent.
