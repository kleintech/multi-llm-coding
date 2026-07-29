# Architecture decision records

Short records for the load-bearing choices — the ones where a reasonable person would pick
differently, and where reversing later would be expensive.

| # | Decision | Why it's load-bearing |
| --- | --- | --- |
| [0001](0001-deterministic-orchestration.md) | Deterministic orchestration; LLMs only at the leaves | Determines reproducibility, cost predictability, and whether the tool needs a privileged vendor to run |
| [0002](0002-litellm-default-backend.md) | LiteLLM as default backend, behind a provider interface | Determines what "vendor-neutral" actually means in code |
| [0003](0003-family-not-model-as-quorum-unit.md) | Model *family*, not model, is the quorum unit | Without it, the tool reports independence it does not have |
| [0004](0004-execution-over-consensus.md) | Execution outranks consensus | The difference between this and a prompt that asks three models |
| [0005](0005-precision-over-recall.md) | Precision over recall, enforced by hard caps | Determines whether anyone leaves the tool switched on |
| [0006](0006-context-sufficiency.md) | Absence claims are a routing problem, not a context-size problem | Addresses the failure mode that kills naive multi-model review |
| [0007](0007-read-only-agent-leaves.md) | Read-only agent leaves for audit mode | Amends ADR-0001; the only place models get tools, and it changes the security justification |

Format: Context / Decision / Rationale / Consequences. Consequences must include what the decision
costs — an ADR that only lists benefits hasn't finished thinking.

Add an ADR when a choice is hard to reverse, when two contributors could reasonably disagree, or
when the reasoning will be non-obvious in six months. Supersede rather than edit.
