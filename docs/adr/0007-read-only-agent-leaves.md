# ADR-0007: Read-only agent leaves for audit mode

**Status:** Accepted (design phase) · **Amends:** [ADR-0001](0001-deterministic-orchestration.md)

## Context

[ADR-0001](0001-deterministic-orchestration.md) makes model calls single-turn and tool-less: the
packet builder pre-computes context, the model answers once. That works for diff review because the
diff bounds what context is relevant, so it *can* be pre-computed.

[Audit mode](../AUDIT_MODE.md) breaks that premise. Asked "does the code still satisfy ADR-0003?",
nothing can pre-compute the answer's evidence — which files matter depends on what the reviewer
finds as it reads. A 128K-LOC repository cannot be handed over wholesale, and a fixed pre-selection
either misses the violation or drowns the reviewer.

## Decision

In audit mode, each reviewer is an **agent with a read-only toolset** — `grep`, `read`, `list` —
under a hard call budget (default 40), given the repo map and one claim.

The orchestrator does **not** change. It still deterministically fans out claims × models, collects,
clusters, and adjudicates. Only the interior of a leaf becomes adaptive.

Constraints that make this acceptable:

- **Read-only.** No writes, no shell, no network. Evidence gathering only.
- **Hard call budget**, so worst-case cost stays computable before dispatch as
  `claims × models × budget × max_tokens` — the ADR-0001 property that actually matters.
- **Reproducible structure.** Same claims and same panel produce the same run shape; clustering,
  adjudication, and rendering remain pure functions over the run record, so offline replay survives.
- **Full tool-call transcripts in the run record**, so a finding can be traced to the searches that
  produced it.
- **Audit mode only.** `crossexam review` keeps tool-less reviewers.

## Rationale

- ADR-0001 anticipated exactly this: *"If adaptive investigation later proves valuable, it belongs
  inside a leaf — a persona given a read-only repo-search tool — not in the orchestrator."* This is
  that case, arriving for a mode that did not exist when the ADR was written.
- The property ADR-0001 protects is **bounded, reproducible orchestration**, not "models never have
  tools." A budgeted read-only leaf preserves both.
- The alternative — pre-computing context for an unbounded question — is not conservative, it is
  wrong. It would produce confident findings from an arbitrary slice, which is the
  context-starvation failure this project exists to avoid.

## Consequences

- **The security posture changes, and the justification is the target, not the tools.**
  `SECURITY.md` leans on tool-less reviewers because PR content is attacker-controlled. Audit mode
  runs against **your own repository**, so that threat model does not apply. Pointing audit mode at
  an untrusted dependency's source reintroduces it, and the docs say so plainly. Read-only, no
  network, no writes are the standing limits regardless.
- **Cost variance rises.** Two reviewers on the same claim may spend 5 calls or 40. The ceiling is
  fixed; the mean is not, which makes per-run estimates less tight than in review mode.
- **Determinism weakens from "same structure and same calls" to "same structure."** Offline replay
  of clustering and adjudication still works, since those consume the recorded findings. Replaying a
  *leaf* does not reproduce its exploration path.
- **Budget exhaustion needs a verdict, not silence.** A reviewer that hits 40 calls without
  concluding returns `INCONCLUSIVE` with its transcript, and that is reported. Treating exhaustion
  as "no findings" would silently convert a timeout into a clean bill of health — the same class of
  error as a jsdom refutation of a layout bug.
- If the bench later shows budgeted exploration also helps *diff* review, extending it there is a
  config change, not a redesign. It should be measured before it is assumed.
