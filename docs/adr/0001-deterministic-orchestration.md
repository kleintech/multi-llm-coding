# ADR-0001: Deterministic orchestration, LLMs only at the leaves

**Status:** Accepted (design phase)

## Context

The pipeline could be driven by an agent — give a capable model the diff, a set of tools, and let
it decide whom to consult, when to dig deeper, and what to publish. That is the fashionable shape
and it is genuinely more flexible.

## Decision

The pipeline is a fixed state machine in Python. Models are called only for leaf tasks: review a
diff under a persona, refute a finding. Fan-out, clustering, verification sequencing, adjudication,
and rendering contain no model calls.

## Rationale

- **Reproducibility.** The eval bench requires that the same input produce the same *run structure*.
  An agentic orchestrator that consults three reviewers today and five tomorrow makes offline
  replay and ablation meaningless — and the bench is how every other decision gets made.
- **Cost predictability.** Worst-case run cost is computable before the first call. An agent loop
  cannot promise a ceiling, and this tool runs on every PR in a public repo where anyone can
  trigger it.
- **Vendor neutrality.** An agentic orchestrator needs a strong model with reliable tool use,
  which in practice means a specific vendor's model as the runtime. That directly contradicts the
  project's premise. Deterministic orchestration means the tool runs with *zero* calls from any
  privileged vendor.
- **Debuggability.** When a bad finding ships, the failure is in a named stage with recorded
  inputs, not in a reasoning trace.
- **The orchestration is genuinely simple.** Fan out, cluster, verify, rank, render. This is not a
  problem that needs judgment; it needs to run the same way every time.

## Consequences

- No adaptive depth: crossexam cannot decide to "look harder" at a suspicious area. Mitigated by
  the persona matrix, which pre-allocates attention, and by packet tiers.
- Clustering uses embeddings + line overlap rather than an LLM. Cheaper and it fails *visibly*
  (duplicate comments) rather than invisibly (silently merged and dropped findings).
- If adaptive investigation later proves valuable, it belongs inside a leaf — a persona given a
  read-only repo-search tool — not in the orchestrator. That keeps the state machine intact and
  the cost bounded per leaf.
